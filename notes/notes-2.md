## chapter3

flex 可以识别 regex, bison 则可以识别语法。书里这一节加入了语法, 也添加了定义变量、赋值等功能，搞得非常牛逼。

bison 采取 `LR(1)` 和类似的 `LALR(1)`, 也有 GLR 的语法。

歧义：对于有效输入，有多余一种可能的 ast。

此外，bison 能够处理的语法，需要值扫描一个 token，书里给了下面一个反例：

```
phrase: cart_animal AND CART
	| work_animal AND PLOW
	
cart_animal: HORSE | GOAT
work_animal: HORSE | OX
```

上面拿到了一个 `HORSE AND` 就不知道怎么规约了，尽管这个是没有歧义的。

那么，处理的方式有：

```
phrase: cart_animal CART
    | work_animal PLOW

cart_animal: HORSE | GOAT
work_animal: HORSE | OX
```

### ast 实现计算器

fb3-1 开始试图用 ast 写个计算器。这里演示了 bison 基本的交互流程。

首先要注意的是，之前我们在 flex 计算器里面把 header 定义在了 `%{ %}` 的 block 里面。

这次 fb3-1 定义了 `fb3-1.h`, 然后在 lex 和 bison 都引入了它。

```c
extern int yylineno; /* from lexer */
void yyerror(char *s, ...);

/* nodes in the Abstract Syntax Tree */
struct ast {
  // 你们写 C 的都不定义好 nodetype 吗...?
  int nodetype;
  struct ast *l;
  struct ast *r;
};

struct numval {
  int nodetype;			/* type K */
  double number;
};

/* build an AST */
struct ast *newast(int nodetype, struct ast *l, struct ast *r);
struct ast *newnum(double d);

/* evaluate an AST */
double eval(struct ast *);

/* delete and free an AST */
void treefree(struct ast *);
```

在 flex 里面，这里只会返回 `NUMBER`, 它实际上不属于两者里面任何一个 type。

那么我们看看 bison 中的定义

```c

%{
#  include <stdio.h>
#  include <stdlib.h>
#  include "fb3-1.h"
%}

%union {
  struct ast *a;
  double d;
}

/* declare tokens */
%token <d> NUMBER
%token EOL

%type <a> exp factor term

%%
```

1. `NUMBER` 类型是一个 `d`, 即 double
2. `exp` `factor` 和 `term` 是一个 `ast` 类型,  number 会被很恶心的转成 `ast`

```c
/* nodes in the Abstract Syntax Tree */
struct ast {
  // 你们写 C 的都不定义好 nodetype 吗...?
  int nodetype;
  struct ast *l;
  struct ast *r;
};

struct numval {
  int nodetype;			/* type K */
  double number;
};
```

那我们可以看到，bison 里面 `$$` 类型就比较好描述了：

```c
%%

calclist: /* nothing */
| calclist exp EOL {
     printf("= %4.4g\n", eval($2));
     treefree($2);
     printf("> ");
 }

 | calclist EOL { printf("> "); } /* blank line or a comment */
 ;

exp: factor
 | exp '+' factor { $$ = newast('+', $1,$3); }
 | exp '-' factor { $$ = newast('-', $1,$3);}
 ;

factor: term
 | factor '*' term { $$ = newast('*', $1,$3); }
 | factor '/' term { $$ = newast('/', $1,$3); }
 ;

term: NUMBER   { $$ = newnum($1); }
 | '|' term    { $$ = newast('|', $2, NULL); }
 | '(' exp ')' { $$ = $2; }
 | '-' term    { $$ = newast('M', $2, NULL); }
 ;
%%

```

### 指定优先级

我们之前做优先级结合，完全是靠构造不同层次的对象，比如 `exp`, `factor`, `term` 来处理的。这样肯定没问题，当然 bison 还支持处理优先级：

```
exp: exp CMP exp          { $$ = newcmp($2, $1, $3); }
   | exp '+' exp          { $$ = newast('+', $1,$3); }
   | exp '-' exp          { $$ = newast('-', $1,$3);}
   | exp '*' exp          { $$ = newast('*', $1,$3); }
   | exp '/' exp          { $$ = newast('/', $1,$3); }
   | '|' exp              { $$ = newast('|', $2, NULL); }
   | '(' exp ')'          { $$ = $2; }
   | '-' exp %prec UMINUS { $$ = newast('M', $2, NULL); }
   | NUMBER               { $$ = newnum($1); }
   | FUNC '(' explist ')' { $$ = newfunc($1, $3); }
   | NAME                 { $$ = newref($1); }
   | NAME '=' exp         { $$ = newasgn($1, $3); }
   | NAME '(' explist ')' { $$ = newcall($1, $3); }
;
```

你看，这个就像之前说的一样，有冲突了。怎么让这个 work 的呢：

```
%nonassoc <fn> CMP
%right '='
%left '+' '-'
%left '*' '/'
%nonassoc '|' UMINUS
```

bison 遇到冲突的时候，利用上面的规则/优先级来处理：

1. 下面的比下面的优先结合，(注： `UMINUS` 是一个伪记号，表示单目 `-`）
2. `nonassoc` 和 `right` 还有 `left` 指定了结合优先级。

后面有个这样的定义：

```
exp: exp CMP exp          { $$ = newcmp($2, $1, $3); }
   | exp '+' exp          { $$ = newast('+', $1,$3); }
   | exp '-' exp          { $$ = newast('-', $1,$3);}
   | exp '*' exp          { $$ = newast('*', $1,$3); }
   | exp '/' exp          { $$ = newast('/', $1,$3); }
   | '|' exp              { $$ = newast('|', $2, NULL); }
   | '(' exp ')'          { $$ = $2; }
   | '-' exp %prec UMINUS { $$ = newast('M', $2, NULL); }
   | NUMBER               { $$ = newnum($1); }
   | FUNC '(' explist ')' { $$ = newfunc($1, $3); }
   | NAME                 { $$ = newref($1); }
   | NAME '=' exp         { $$ = newasgn($1, $3); }
   | NAME '(' explist ')' { $$ = newcall($1, $3); }
```

`-` 右比较低的优先级，但是 `%prec UMINUS` 给了它 `UMINUS` 的优先级。

（我有点忘了编译原理了，有空想看下 parsing techniques）

### 让我们来看看这个非常复杂的程序

```
%union {
  struct ast *a;
  double d;
  struct symbol *s;		/* which symbol */
  struct symlist *sl;
  int fn;			/* which function */
}

%type <a> exp stmt list explist
```



```
stmt: IF exp THEN list           { $$ = newflow('I', $2, $4, NULL); }
   | IF exp THEN list ELSE list  { $$ = newflow('I', $2, $4, $6); }
   | WHILE exp DO list           { $$ = newflow('W', $2, $4, NULL); }
   | exp
;

list: /* nothing */ { $$ = NULL; }
   | stmt ';' list { if ($3 == NULL)
	                $$ = $1;
                      else
			$$ = newast('L', $1, $3);
                    }
   ;
```

1. 这里区分了语句 `stmt` 和表达式 `exp`, stmt 是一个控制流或者表达式
2. List 是一个右递归的定义，定义成 `stmt ; list`, 这个每次规约一个 `stmt` 处理，容易从头到位创建 list (但其实我也不是特别懂)

`calclist` 建立在这两个东西上面:

```
%start calclist

%%
calclist: /* nothing */
  | calclist stmt EOL {
    if(debug) dumpast($2, 0);
     printf("= %4.4g\n> ", eval($2));
     treefree($2);
    }
  | calclist LET NAME '(' symlist ')' '=' list EOL {
                       dodef($3, $5, $8);
                       printf("Defined %s\n> ", $3->name); }

  | calclist error EOL { yyerrok; printf("> "); }
 ;
%%

```

这里还保存了函数定义。然后 `error` 是为了将函数恢复到可用的状态。error 详见：

1. https://www.gnu.org/software/bison/manual/html_node/Error-Recovery.html
2. https://www.gnu.org/software/bison/manual/html_node/Error-Reporting-Function.html

