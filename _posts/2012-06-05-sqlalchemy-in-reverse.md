---
description: |
 When using data in a relational database you often use an Object Relational
  Mapper (ORM) so you can pretend that you are only dealing with standard
  objects instead of having to deal with SQL directly. The ORM will then
  generate SQL automatically for you, and magically turn query results into
  objects again. For a recent project I had to do the exact reverse: turn
  a SQL expression into a SQLAlchemy construct.

  Doing this required building a parser for SQL expressions to generate
  a syntax tree, and then converting the syntax tree into a SQLAlchemy
  expression.
---
A large part of my dayjob involves writing web applications. We deal with a
reasonable amount of data, most of which is stored in a
[PostgreSQL](www.postgresql.org) database. Object Relational Mappers (ORMs) are
a great tool to help with that: they allow you to forget all about SQL, instead
allowing you pretend you are just dealing with standard objects.  And if you
are using Python chances you you will be using the swiss army knife of SQL for
python: [SQLAlchemy](http://www.sqlalchemy.org/). Perhaps I should say swiss
army battallion - SQLAlchemy is just that good. Recently I had an interesting
challenge though: instead of using the SQLALchemy ORM to generate SQL I had to
do the opposite: turn a SQL expression into a SQLAlchemy expression. Doing this
involved two challenges: parsing SQL, and turning the result into a SQLAlchemy
expression.

Step 1: Parsing SQL
-------------------

Luckily my problem was fairly constrained: I did not need to parse all SQL
constructs, but only basic expressions that check a single row and do not
involve subqueries.  An example of a expression I have to handle is this:
```(price<20 OR sale='t') AND colour='black'```, which should match articles
that are either cheap or on sale and come in a very fashionable black. Parsing
something like ```id in (SELECT id FROM othertable WHERE brand='acme') AND
price<20``` is out of scope luckily. With this constraint it is not to hard to
write down a simple grammar and feed that to a parser generator. I ended up
picking [LEPL](http://www.acooke.org/lepl/) because unlike many other toolkits
its documentation was understandable, and its syntax is reasonably similar to
standard BNF. The resulting parser definition looks like this:


```python
from lepl import *

class Negation(Node):
    pass

class ParensExpression(Node):
    pass

class Expression(Node):
    pass

class BoolList(Node):
    pass

class ValueList(Node):
    pass

comma = Drop(',')
string = String(quote="'")
integer = Integer() >> int
real = Regexp(r'[+-]?[0-9]+(?:\.[0-9]*)?') >> decimal.Decimal
column = Word(Letter() | '_', Letter() | '_' | Digit()) > 'column'
operation = Literal('=') | Literal('!=') | \
                Literal('<') | Literal('<=') | \
                Literal('>') | Literal('>=') > 'operator'
not_keyword = Regexp('[Nn][Oo][Tt]')
in_keyword = Regexp('[Ii][Nn]')
whitespace = ~Whitespace()[1:]
in_operator = (not_keyword & whitespace)[:1] & in_keyword > 'operator'
logic_operator = Regexp('[Aa][Nn][Dd]') | Regexp('[Oo][Rr]') > 'operator'

value = string | integer | real

with DroppedSpace():
    expression = Delayed()
    value_list = Drop('(') & value[:, comma] & Drop(')') > ValueList
    comparison = column & operation & value > Expression
    one_of = column & in_operator  & value_list > Expression
    parens = Drop('(') & expression & Drop(')') > ParensExpression

    single_expression = Delayed()
    negation = not_keyword & single_expression > Negation
    single_expression += negation | comparison | one_of | parens
    logic_expression = single_expression & logic_operator & expression > BoolList
    expression += logic_expression | single_expression
    expression.config.auto_memoize()
    sql_expression = expression & Eos()

```

This is probably not the most optimal spelling  ; in particular the
way I did case insenstivity for keywords feels quite sub-optimal. Even
so it does produce a lovely ```sql_expression``` parser that you can use to
parse things:

```
>>> print sql_expression.parse("price<20 OR sale='t'")[0]
BoolList
 +- Expression
 |   +- column 'price'
 |   +- operator '<'
 |   `- 20
 +- operator 'OR'
 `- Expression
     +- column 'sale'
     +- operator '='
     `- 't'
```

The output from parsing a string is an abstract syntax tree for the SQL
expression. Part one accomplished!


Step 2: generate SQLAlchemy expression
--------------------------------------

The next step is turning the ast into a SQLAlchemy expression. In order to keep
this simple (and easier to test) it makes sense to split this into two
compoments: building an expression for a simple operation (for exampe *price
below 10 euro*), and then building the expressions from those operations.


```python
_OPERATIONS = {
        '=': operator.eq,
        '!=': operator.ne,
        '<': operator.lt,
        '<=': operator.le,
        '>': operator.gt,
        '>=': operator.ge,
        'IN': operator.contains,
        'NOT IN': lambda a, b: not operator.contains(b, a)
        }


def apply_operation(column, operation, value, check_type):
    """Apply an operation against a SQL attribute.

    :param column: column to apply operation to
    :type column: sqlalchemy.orm.attributes.InstrumentedAttribute
    :param str operation: operator to apply
    :param value: value(s) to compare column with
    :param check_type: tuple of allowed value types
    :return: SQLalchemy expression instance

    The check_type is used to make sure no invalid value types are used:

    >>> print apply_operation(Article.type, '=', 'coat', basestring)
    article.type = :type_1

    The check_type is used to make sure no invalid value types are used:

    >>> apply_operation(Article.type, '=', 5, basestring)
    TypeError: Bad value type
    """
    if operation in ['IN', 'NOT IN']:
        if not all(isinstance(v, check_type) for v in value):
            raise TypeError('Bad value type')
        op = column.in_(value)
        if operation == 'NOT IN':
            op = sql.not_(op)
        return op
    else:
        if not isinstance(value, check_type):
            raise TypeError('Bad value type')
        return _OPERATIONS[operation](column, value)
```

this takes advantage of the fact that SQLAlchemy uses normal python
operations (pulled in here through the
[operator](http://docs.python.org/library/operator) module) and turns
those into its own [ColumnElement](http://docs.sqlalchemy.org/en/latest/core/expression_api.html#sqlalchemy.sql.expression.ColumnElement). The next
step is turning an entire ast into a SQLAlchemy construct:

```python
_BOOL_OPERATIONS = {
        'AND': sql.and_,
        'OR': sql.or_,
        }

def convert_ast_to_sql_expression(ast):
    """Convert an abstract syntax tree to a SQLAlchemy expression.

    :param ast: Abstract Syntax Tree
    :return: SQLAlchemy expression

    This function takes an abstract syntax tree (AST) as generated by the
    LEPL parser :py:obj:`sql_expression` and converts it to an SQLAlchemy
    expression that can be used to select articles.
    """
    while isinstance(ast, ParensExpression):
        ast = ast[0]

    if isinstance(ast, BoolList):
        try:
            operation = _BOOL_OPERATIONS[' '.join(ast.operator).upper()]
        except KeyError:
            raise RuntimeError('Unknown operator found in AST')
        left = convert_ast_to_sql_expression(ast[0])
        right = convert_ast_to_sql_expression(ast[2])
        return operation(left, right)
    elif isinstance(ast, Expression):
        column = ast.column[0]
        operation = ' '.join(ast.operator).upper()
        value = ast[2]
        if column == 'id':
            return apply_operation(Article.id, operation, value, int)
        elif column == 'price':
            return apply_operation(Article.price, operation, value,
                    (int, decimal.Decimal))
        else:
            return apply_operation(getattr(Article, column), operation, value,
                    basestring)
    elif isinstance(ast, Negation):
        return sql.not_(convert_ast_to_sql_expression(ast[0]))
    else:
        raise RuntimeError('Unexpected AST node type')
```

Step 3: a simple API
--------------------

With all the base logic in place the last step was creating a simple
function to do what I initially set out to do: convert a string with
a SQL expression to a SQLAlchemy expression.

```python
def sql_to_sqlalchemy(flt):
    """Convert a SQL expression to a SQLAlchemy query.

    :param unicode flt: SQL expression
    :return: SQLAlchemy expression
    """
    if not flt:
        return None

    try:
        ast = sql_expression.parse(flt)[0]
    except FullFirstMatchException:
        raise ValueError('Invalid SQL expression.')

    return convert_ast_to_sql_expression(ast)
```
