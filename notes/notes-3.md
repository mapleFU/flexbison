## SQL 处理

这里用 RPN 来处理 SQL，我们有 `SELECT a, b, c from tbl`, 那么 RPN 可以表示成：

```
column a
column b
column c
table tbl
Select 3 (表示 select 3 个 column)
```

SQL 需要处理很多关键字，书上代码列了五页，这里大部分就硬编码就行了，此外有一些特殊的，比如 `AND`, 下面两种场景里 `AND` 含义不同

### flex

```
IF (a && b, ...)
IF (a AND b, ...)

BETWEEN c AND d..
```

实际上 BETWEEN 会在 flex 切成一个特殊的模式。在使用 LALR 的时候，下面被归为同一个:

```
EXISTS	{ yylval.subtok = 0; return EXISTS; }
NOT[ \t]+EXISTS	{ yylval.subtok = 1; return EXISTS; }
```

实际上，采用 GLR 的时候，就可以处理更强的情况了。bison LALR 只能向前查看一个 token。

同时，字符串，数字可以用 `STRING`,`INTNUM`, `APPROXNUM`

### bison

定义了基本规则后，这里也定义了 `union` 等

我们先来看看表达式，即 `expr` 部分：

```
   /**** expressions ****/

expr: NAME          { emit("NAME %s", $1); free($1); }
   | USERVAR         { emit("USERVAR %s", $1); free($1); }
   | NAME '.' NAME { emit("FIELDNAME %s.%s", $1, $3); free($1); free($3); }
   | STRING        { emit("STRING %s", $1); free($1); }
   | INTNUM        { emit("NUMBER %d", $1); }
   | APPROXNUM     { emit("FLOAT %g", $1); }
   | BOOL          { emit("BOOL %d", $1); }
   ;

expr: expr '+' expr { emit("ADD"); }
   | expr '-' expr { emit("SUB"); }
   | expr '*' expr { emit("MUL"); }
   | expr '/' expr { emit("DIV"); }
   | expr '%' expr { emit("MOD"); }
   | expr MOD expr { emit("MOD"); }
   | '-' expr %prec UMINUS { emit("NEG"); }
   | expr ANDOP expr { emit("AND"); }
   | expr OR expr { emit("OR"); }
   | expr XOR expr { emit("XOR"); }
   | expr COMPARISON expr { emit("CMP %d", $2); }
   | expr COMPARISON '(' select_stmt ')' { emit("CMPSELECT %d", $2); }
   | expr COMPARISON ANY '(' select_stmt ')' { emit("CMPANYSELECT %d", $2); }
   | expr COMPARISON SOME '(' select_stmt ')' { emit("CMPANYSELECT %d", $2); }
   | expr COMPARISON ALL '(' select_stmt ')' { emit("CMPALLSELECT %d", $2); }
   | expr '|' expr { emit("BITOR"); }
   | expr '&' expr { emit("BITAND"); }
   | expr '^' expr { emit("BITXOR"); }
   | expr SHIFT expr { emit("SHIFT %s", $2==1?"left":"right"); }
   | NOT expr { emit("NOT"); }
   | '!' expr { emit("NOT"); }
   | USERVAR ASSIGN expr { emit("ASSIGN @%s", $1); free($1); }
   ;    

expr:  expr IS NULLX     { emit("ISNULL"); }
   |   expr IS NOT NULLX { emit("ISNULL"); emit("NOT"); }
   |   expr IS BOOL      { emit("ISBOOL %d", $3); }
   |   expr IS NOT BOOL  { emit("ISBOOL %d", $4); emit("NOT"); }
   ;

expr: expr BETWEEN expr AND expr %prec BETWEEN { emit("BETWEEN"); }
   ;


val_list: expr { $$ = 1; }
   | expr ',' val_list { $$ = 1 + $3; }
   

opt_val_list: /* nil */ { $$ = 0; }
   | val_list
   ;

expr: expr IN '(' val_list ')'       { emit("ISIN %d", $4); }
   | expr NOT IN '(' val_list ')'    { emit("ISIN %d", $5); emit("NOT"); }
   | expr IN '(' select_stmt ')'     { emit("INSELECT"); }
   | expr NOT IN '(' select_stmt ')' { emit("INSELECT"); emit("NOT"); }
   | EXISTS '(' select_stmt ')'      { emit("EXISTS"); if($1)emit("NOT"); }
   ;

expr: NAME '(' opt_val_list ')' {  emit("CALL %d %s", $3, $1); free($1); }
   ;

  /* functions with special syntax */
expr: FCOUNT '(' '*' ')' { emit("COUNTALL"); }
   | FCOUNT '(' expr ')' { emit(" CALL 1 COUNT"); } 

expr: FSUBSTRING '(' val_list ')' {  emit("CALL %d SUBSTR", $3);}
   | FSUBSTRING '(' expr FROM expr ')' {  emit("CALL 2 SUBSTR"); }
   | FSUBSTRING '(' expr FROM expr FOR expr ')' {  emit("CALL 3 SUBSTR"); }
| FTRIM '(' val_list ')' { emit("CALL %d TRIM", $3); }
   | FTRIM '(' trim_ltb expr FROM val_list ')' { emit("CALL 3 TRIM"); }
   ;

trim_ltb: LEADING { emit("INT 1"); }
   | TRAILING { emit("INT 2"); }
   | BOTH { emit("INT 3"); }
   ;

expr: FDATE_ADD '(' expr ',' interval_exp ')' { emit("CALL 3 DATE_ADD"); }
   |  FDATE_SUB '(' expr ',' interval_exp ')' { emit("CALL 3 DATE_SUB"); }
   ;

interval_exp: INTERVAL expr DAY_HOUR { emit("NUMBER 1"); }
   | INTERVAL expr DAY_MICROSECOND { emit("NUMBER 2"); }
   | INTERVAL expr DAY_MINUTE { emit("NUMBER 3"); }
   | INTERVAL expr DAY_SECOND { emit("NUMBER 4"); }
   | INTERVAL expr YEAR_MONTH { emit("NUMBER 5"); }
   | INTERVAL expr YEAR       { emit("NUMBER 6"); }
   | INTERVAL expr HOUR_MICROSECOND { emit("NUMBER 7"); }
   | INTERVAL expr HOUR_MINUTE { emit("NUMBER 8"); }
   | INTERVAL expr HOUR_SECOND { emit("NUMBER 9"); }
   ;

expr: CASE expr case_list END           { emit("CASEVAL %d 0", $3); }
   |  CASE expr case_list ELSE expr END { emit("CASEVAL %d 1", $3); }
   |  CASE case_list END                { emit("CASE %d 0", $2); }
   |  CASE case_list ELSE expr END      { emit("CASE %d 1", $2); }
   ;

case_list: WHEN expr THEN expr     { $$ = 1; }
         | case_list WHEN expr THEN expr { $$ = $1+1; } 
   ;

expr: expr LIKE expr { emit("LIKE"); }
   | expr NOT LIKE expr { emit("LIKE"); emit("NOT"); }
   ;

expr: expr REGEXP expr { emit("REGEXP"); }
   | expr NOT REGEXP expr { emit("REGEXP"); emit("NOT"); }
   ;

expr: CURRENT_TIMESTAMP { emit("NOW"); }
   | CURRENT_DATE	{ emit("NOW"); }
   | CURRENT_TIME	{ emit("NOW"); }
   ;

expr: BINARY expr %prec UMINUS { emit("STRTOBIN"); }
   ;
```

1. 最简单的是处理 变量名和常量，这里包含了 `table.name` 和 `@` 的常量字符串等，这里只会发射对应 RPN
2. `+` `-` `OR` 等会 emit 出来，再次构成 `expr`
   1. 同时，可能有递归的 `expr COMPARISON '(' select_stmt ')'` 等表达式
   2. 和 `IS NULL` 或者 `IS BOOL` 这些 is 的指涉
3. 用 `val_list` 的语法 （这个是变长参数列表），处理多个参数，例如 `IN`, `BETWEEN .. AND`；对于 `val_list`, 需要有额外的方式记录 "已经有多少个了"，然后处理这个数目。
4. 函数会打一个 `emit("call 函数要从栈上拿的参数数目 func_name")`, 有的类型转化也走这个了

#### select 语句

```
stmt: select_stmt { emit("STMT"); }
   ;

select_stmt: SELECT select_opts select_expr_list
                        { emit("SELECTNODATA %d %d", $2, $3); } ;
    | SELECT select_opts select_expr_list
     FROM table_references
     opt_where opt_groupby opt_having opt_orderby opt_limit
     opt_into_list { emit("SELECT %d %d %d", $2, $3, $5); } ;
;
```

1. `stmt` 产生 `emit("STMT")` 作为分隔符
2. `select` 的结构如 `select_stmt` 表示，第一个部分 `SELECTNODATA`, 第二部分需要访问表。

select 的表引用部分非常复杂, 可能是一个 table, 可能是多个，可能有 JOIN, 没准还有 subquery：

```
table_references:    table_reference { $$ = 1; }
    | table_references ',' table_reference { $$ = $1 + 1; }
    ;

table_reference:  table_factor
  | join_table
;

table_factor:
    NAME opt_as_alias index_hint { emit("TABLE %s", $1); free($1); }
  | NAME '.' NAME opt_as_alias index_hint { emit("TABLE %s.%s", $1, $3);
                               free($1); free($3); }
  | table_subquery opt_as NAME { emit("SUBQUERYAS %s", $3); free($3); }
  | '(' table_references ')' { emit("TABLEREFERENCES %d", $2); }
  ;

```

而 JOIN Table 则表示了 Join:

```
join_table:
    table_reference opt_inner_cross JOIN table_factor opt_join_condition
                  { emit("JOIN %d", 0100+$2); }
  | table_reference STRAIGHT_JOIN table_factor
                  { emit("JOIN %d", 0200); }
  | table_reference STRAIGHT_JOIN table_factor ON expr
                  { emit("JOIN %d", 0200); }
  | table_reference left_or_right opt_outer JOIN table_factor join_condition
                  { emit("JOIN %d", 0300+$2+$3); }
  | table_reference NATURAL opt_left_or_right_outer JOIN table_factor
                  { emit("JOIN %d", 0400+$3); }
  ;

```