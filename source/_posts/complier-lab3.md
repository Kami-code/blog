---
title: complier-lab3
date: 2021-11-04 14:41:28
tags: [compliers, lab]
---

​	This lab comes from SE3355 ***Compilers\***  lab4, which requires me to use [Bisonc++](https://fbb-git.gitlab.io/bisoncpp/manual/bisonc++.html) to implement a parser for the Tiger language.

<!-- more -->

​	The blog is written after the completion of the lab, so some details may be forgotten. Our main target of this lab is to use a sequence of tokens which the lab3's lexical scanner gives, to parse them and create a AST(Abstract Syntax Tree). The AST representation is necessary for us to do the semantic type checking in the next lab. What's more, we need to convert AST to a intermediate language then, for our final processing when trying to get assembly language and binary executable file.

​	For convenience and succinctness, I will not again to explain the basic structure and class of the AST in the code. The code will be provided in my github repo when all labs are finished.

​	If you are not familiar with the workflow of the Bison C++, refer to the [documentation](https://fbb-git.gitlab.io/bisoncpp/manual/bisonc++.html)  for more details.

## Part 1 Some Simple Productions

According to Appendix A of tiger book, we can easily write down the following productions.

### 1.1 basic productions of expression

​	The basic primitive expression constructor is necessary, which forms the lower-level nodes(mainly leaf nodes) of the AST.

```c++
exp : INT {$$ = new absyn::IntExp(scanner_.GetTokPos(), $1);}
    | MINUS exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::MINUS_OP, new absyn::IntExp(scanner_.GetTokPos(), 0), $2);}
    | STRING {$$ = new absyn::StringExp(scanner_.GetTokPos(), $1);}
    | lvalue {$$ = new absyn::VarExp(scanner_.GetTokPos(), $1);}
    | NIL {$$ = new absyn::NilExp(scanner_.GetTokPos());}
;
```

### 1.2 Operation productions of expression

```c++
exp : MINUS INT {$$ = new absyn::IntExp(scanner_.GetTokPos(), -1 * $2);}
    | exp PLUS exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::PLUS_OP, $1, $3);}
    | exp MINUS exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::MINUS_OP, $1, $3);}
    | exp TIMES exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::TIMES_OP, $1, $3);}
    | exp DIVIDE exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::DIVIDE_OP, $1, $3);}
    | exp AND exp {$$ = new absyn::IfExp(scanner_.GetTokPos(), $1, $3, new absyn::IntExp(scanner_.GetTokPos(), 0));}
    | exp OR exp {$$ = new absyn::IfExp(scanner_.GetTokPos(), $1, new absyn::IntExp(scanner_.GetTokPos(), 1), $3);}
    | exp EQ exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::EQ_OP, $1, $3);}
    | exp NEQ exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::NEQ_OP, $1, $3);}
    | exp LT exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::LT_OP, $1, $3);}
    | exp LE exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::LE_OP, $1, $3);}
    | exp GT exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::GT_OP, $1, $3);}
    | exp GE exp {$$ = new absyn::OpExp(scanner_.GetTokPos(), absyn::GE_OP, $1, $3);}
;
```

​	We need to highlight the implementation of the AND and OR operation. Since we don't have a AND_OP and OR_OP in the predefined declarations of the AST. We can use the following conversion:

```
exp1 AND exp2
<=>
if exp1 then exp2 else 0
```

```
exp1 OR exp2
<=>
if exp1 then 1 else exp2
```

​	Since in tiger language, the if-expression will return either the then-body or the else-body according to the condition, we can implement the conversion without any worry.

### 1.3 General Declarations

```c++
exp : LET decs IN expseq END {$$ = new absyn::LetExp(scanner_.GetTokPos(), $2, $4);};

decs : decs_nonempty {$$ = $1;}
    |   {$$ = new absyn::DecList();} //may be empty here
;

decs_nonempty : decs_nonempty_s decs {$$ = $2; $$->Prepend($1);};

decs_nonempty_s : vardec {$$ = $1;}
    | tydec {$$ = new absyn::TypeDec(scanner_.GetTokPos(), $1);}
    | fundec {$$ = new absyn::FunctionDec(scanner_.GetTokPos(), $1);}
;
```

​	In let-in-end format, tiger language allow empty declarations in production. So decs can be devided into empty one and non-empty one. For those are not empty, it can be forwardly inferred to a list of decs_nonempty_s. We use a DecList class to realize such behavior. For example, 

```
some declaration;
some declaration;
some declaration;
some declaration;
```

 will finally form a class of absyn::DecList with four declarations in its member variable dec_list_ whose type is std::List<dec *>.

### 1.4 Variable declaration

```c++
vardec : VAR ID ASSIGN exp {$$ = new absyn::VarDec(scanner_.GetTokPos(), $2, nullptr, $4);}
    | VAR ID COLON ID ASSIGN exp {$$ = new absyn::VarDec(scanner_.GetTokPos(), $2, $4, $6);};
```

​	Since now,  we can handle codes like

```C
var a := 3;
var b : string := "foo";
```

### 1.5 Type declaration

```c++
tydec : tydec_one tydec {$$ = $2; $$->Prepend($1);}
    | tydec_one {$$ = new absyn::NameAndTyList($1);};

tydec_one : TYPE ID EQ ty {$$ = new absyn::NameAndTy($2, $4); };

ty : ID {$$ = new absyn::NameTy(scanner_.GetTokPos(), $1);}
    | LBRACE tyfields RBRACE {$$ = new absyn::RecordTy(scanner_.GetTokPos(), $2);}
    | ARRAY OF ID {$$ = new absyn::ArrayTy(scanner_.GetTokPos(), $3);};

tyfields : tyfields_nonempty {$$ = $1;}
    | {$$ = new absyn::FieldList();} //may be empty here
;

tyfields_nonempty : tyfield {$$ = new absyn::FieldList($1);}
    | tyfield COMMA tyfields_nonempty {$$ = $3; $$->Prepend($1);};

tyfield : ID COLON ID {$$ = new absyn::Field(scanner_.GetTokPos(), $1, $3);};
```

​	As we can see, the tydec also contains a list of tydec_one, so we can apply the same strategy when we handle decs. There are three type declaration in tiger language.

```c++
type a = int;
type b = array of a;
type c = {name:string, score:b};
```

​	So we can easily recognize the first and third line of ty production, but the nested one seems more difficult. Since the typefields is also a non-fixed size list, so we also need to split it into the empty one and non-empty one. Each production will add one element to the typefields.

### 1.7 Function declaration

​	The function declaration consists of two types.

```
function id (tyfields) = exp
function id (tyfields) : type-id = exp
```

​	So we can easily get the following code.

```c++
fundec : fundec_one {$$ = new absyn::FunDecList($1);}
    | fundec_one fundec {$$ = $2; $$->Prepend($1);};

fundec_one : FUNCTION ID LPAREN tyfields RPAREN EQ exp {$$ = new absyn::FunDec(scanner_.GetTokPos(), $2, $4, nullptr, $7);}
    | FUNCTION ID LPAREN tyfields RPAREN COLON ID EQ exp {$$ = new absyn::FunDec(scanner_.GetTokPos(), $2, $4, $7, $9);};
```

### 1.8 Nested type assignment

​	To initialize those type are declared in nest as follows,

```c++
list{first=i, rest=readlist()}
```

​	we can also have

```c++
exp : ID LBRACE rec RBRACE {$$ = new absyn::RecordExp(scanner_.GetTokPos(), $1, $3);}

rec : rec_nonempty {$$ = $1;}
    | {$$ = new absyn::EFieldList();};

rec_nonempty : rec_one {$$ = new absyn::EFieldList($1);}
    | rec_one COMMA rec_nonempty {$$ = $3; $$->Prepend($1);};

rec_one : ID EQ exp {$$ = new absyn::EField($1, $3);};
```

### 1.9 Other productions

​	According to Appendix A, it's easy to understand the following code.

```c++
exp : LPAREN sequencing RPAREN {$$ = new absyn::SeqExp(scanner_.GetTokPos(), $2);}
    | LPAREN exp RPAREN {$$ = $2;}
    | LPAREN RPAREN {$$ = new absyn::VoidExp(scanner_.GetTokPos());}
    | LET IN END {$$ = new absyn::VoidExp(scanner_.GetTokPos());}
    | ID LPAREN actuals RPAREN {$$ = new absyn::CallExp(scanner_.GetTokPos(), $1, $3);}
    | lvalue ASSIGN exp {$$ = new absyn::AssignExp(scanner_.GetTokPos(), $1, $3);}
    | WHILE exp DO exp {$$ = new absyn::WhileExp(scanner_.GetTokPos(), $2, $4);}
    | FOR ID ASSIGN exp TO exp DO exp {$$ = new absyn::ForExp(scanner_.GetTokPos(), $2, $4, $6, $8);}
    | BREAK {$$ = new absyn::BreakExp(scanner_.GetTokPos());};
    
actuals : nonemptyactuals {$$ = $1;}
    | {$$ = new absyn::ExpList();};

nonemptyactuals : exp {$$ = new absyn::ExpList($1);}
    | exp COMMA nonemptyactuals {$$ = $3; $$->Prepend($1);};

sequencing : exp sequencing_exps {$$ = $2; $$->Prepend($1); };

sequencing_exps: SEMICOLON exp sequencing_exps {$$ = $3; $$->Prepend($2);}
    | {$$ = new absyn::ExpList();};

expseq :  {$$ = new absyn::VoidExp(scanner_.GetTokPos());}
    | sequencing {$$ = new absyn::SeqExp(scanner_.GetTokPos(), $1);};
```



## Part 2 Handle shift-reduce conflict

### 2.1 If-then-else conflict

​	I solve the conflict according to [blog](https://stackoverflow.com/questions/12731922/reforming-the-grammar-to-remove-shift-reduce-conflict-in-if-then-else).

```c++
%nonassoc THEN
%nonassoc ELSE
...
exp : IF exp THEN exp ELSE exp {$$ = new absyn::IfExp(scanner_.GetTokPos(), $2, $4, $6);}
    | IF exp THEN exp {$$ = new absyn::IfExp(scanner_.GetTokPos(), $2, $4, nullptr);}
```



### 2.2 ID[exp] conflict

​	I have spent a lot time on debugging this conflict. To be specific, the problem can be illustrated in the following sheet.

```
ID [exp] ·, OF 		shift
ID [exp] ·, $ 		reduce
ID [exp] ·, DOT 	shift
ID [exp] ·, [ 		shift
ID [exp] ·, ASSIGN 	shift
```

​	Let's take an example,

```
var b := intArray [N] of 0
a = c[10]
c[10].first = 5
d[10][5] = 2
row[7] := 1
```

​	I have tried a lot of method from [blog1](https://stackoverflow.com/questions/5381909/bison-shift-reduce-conflict) and [blog2](https://fbb-git.gitlab.io/bisoncpp/manual/bisonc++07.html#l120), but it seems neither of both are useful. To enforce every production do reduce first, I configure "special_one" to achieve my goal.

```c++
exp : special_one OF exp {$$ = new absyn::ArrayExp(scanner_.GetTokPos(), ((absyn::SimpleVar *)(((absyn::SubscriptVar *)($1))->var_))->sym_, ((absyn::SubscriptVar *)($1))->subscript_, $3);}

lvalue : oneormore DOT ID {$$ = new absyn::FieldVar(scanner_.GetTokPos(), $1, $3);}
    | oneormore LBRACK exp RBRACK {$$ = new absyn::SubscriptVar(scanner_.GetTokPos(), $1, $3);}
    | oneormore  {$$ = $1;}
    | special_one DOT ID {$$ = new absyn::FieldVar(scanner_.GetTokPos(), $1, $3);}
    | special_one LBRACK exp RBRACK {$$ = new absyn::SubscriptVar(scanner_.GetTokPos(), $1, $3);}
    | special_one  {$$ = $1;};

special_one : one LBRACK exp RBRACK{$$ = new absyn::SubscriptVar(scanner_.GetTokPos(), $1, $3);};

oneormore : one DOT ID {$$ = new absyn::FieldVar(scanner_.GetTokPos(), $1, $3);}
    | one {$$ = $1;};

one : ID {$$ = new absyn::SimpleVar(scanner_.GetTokPos(), $1);};
```

​	I configured special_one equals "id[exp]", enforcing every "id[exp]" to be reduced to special_one.