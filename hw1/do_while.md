# 參考03b-compiler2.c中的寫法

## main(dump)
```
#include "compiler.h"

int readText(char *fileName, char *text, int size) {
  FILE *file = fopen(fileName, "r");
  int len = fread(text, 1, size, file);
  text[len] = '\0';
  fclose(file);
  return len;
}

void dump(char *strTable[], int top) {
  printf("========== dump ==============\n");
  for (int i=0; i<top; i++) {
    printf("%d:%s\n", i, strTable[i]);
  }
}

int main(int argc, char * argv[]) {
  readText(argv[1], code, TMAX);
  puts(code);
  lex(code);
  dump(tokens, tokenTop);
  parse();
  return 0;
}
```
## lexer(lex)
```
#include "compiler.h"

#define TMAX 10000000
#define LMAX 100

char *typeName[5] = {"Id", "Int", "Keyword", "Literal", "Char"};
char code[TMAX], *p;
char strTable[TMAX], *strTableEnd=strTable;
char *tokens[TMAX], tokenTop=0, tokenIdx=0, token[LMAX];

char *scan() {
  while (isspace(*p)) p++;

  char *start = p;
  int type;
  if (*p == '\0') return NULL;
  if (*p == '"') {
    p++;
    while (*p != '"') p++;
    p++;
    type = Literal;
  } else if (*p >='0' && *p <='9') { // 數字
    while (*p >='0' && *p <='9') p++;
    type = Int;
  } else if (isAlpha(*p) || *p == '_') { // 變數名稱或關鍵字
    while (isAlpha(*p) || isDigit(*p) || *p == '_') p++;
    type = Id;
  } else { // 單一字元
    p++;
    type = Char;
  }
  int len = p-start;
  strncpy(token, start, len);
  token[len] = '\0';
  return token;


void lex(char *code) {
  printf("========== lex ==============\n");
  p = code;
  tokenTop = 0;
  while (1) {
    char *tok = scan();
    if (tok == NULL) break;
    strcpy(strTableEnd, tok);
    tokens[tokenTop++] = strTableEnd;
    strTableEnd += (strlen(tok)+1);
    printf("token=%s\n", tok);
  }
}
```
## compiler(parse)
```
#include <assert.h>
#include "compiler.h"

int E();
void STMT();
void IF();
void BLOCK();

int tempIdx = 0, labelIdx = 0;

#define nextTemp() (tempIdx++)
#define nextLabel() (labelIdx++)
#define emit printf

int isNext(char *set) {
  char eset[SMAX], etoken[SMAX];
  sprintf(eset, " %s ", set);
  sprintf(etoken, " %s ", tokens[tokenIdx]);
  return (tokenIdx < tokenTop && strstr(eset, etoken) != NULL);
}

int isEnd() {
  return tokenIdx >= tokenTop;
}

char *next() {
  // printf("token[%d]=%s\n", tokenIdx, tokens[tokenIdx]);
  return tokens[tokenIdx++];
}

char *skip(char *set) {
  if (isNext(set)) {
    return next();
  } else {
    printf("skip(%s) got %s fail!\n", set, next());
    assert(0);
  }
}

// F = (E) | Number | Id
int F() {
  int f;
  if (isNext("(")) { // '(' E ')'
    next(); // (
    f = E();
    next(); // )
  } else { // Number | Id
    f = nextTemp();
    char *item = next();
    emit("t%d = %s\n", f, item);
  }
  return f;
}

// E = F (op E)*
int E() {
  int i1 = F();
  while(isNext("+ - * / & | ! < > =")){
    char *op = next();
    if(isNext("+ - * / & | ! < > =")){
      char *a = next();
      int i2 = E();
      int i = nextTemp();
      emit("t%d = t%d %s%s t%d\n", i, i1, op, a, i2);
      i1 = i;
    }
    else{
      int i2 = E();
      int i = nextTemp();
      emit("t%d = t%d %s t%d\n", i, i1, op, i2);
      i1 = i;
    }
  }
  return i1;
}

// ASSIGN = id '=' E;
void ASSIGN() {
  char *id = next();
  skip("=");
  int e = E();
  skip(";");
  emit("%s = t%d\n", id, e);
}

//IF = if (E) STMT (else STMT)?
 void IF() {
   int elseBegin = nextLabel();
   int endifLabel = nextLabel();
   skip("if");
   skip("(");
   int e = E();
   skip(")");
   emit("ifnot t%d goto L%d\n", e, elseBegin);
   STMT();
   emit("goto L%d\n", endifLabel);
   if (isNext("else")) {
     emit("(L%d)\n", elseBegin);
     skip("else");
     STMT();
   } 
   emit("(L%d)\n", endifLabel);
 }

void DOWHILE(){
  int whileBegin = nextLabel();
  emit("(L%d)\n", whileBegin);
  skip("do");
  STMT();
  if(isNext("while")){
    skip("while");
    skip("(");
    int e = E();
    emit("if T%d goto L%d\n", e,whileBegin);
    skip(")");
    skip(";");
  }
  
  else{
    emit("Error");
    assert(0);
  }
}

// while (E) STMT
void WHILE() {
  int whileBegin = nextLabel();
  int whileEnd = nextLabel();
  emit("(L%d)\n", whileBegin);
  skip("while");
  skip("(");
  int e = E();
  emit("if not T%d goto L%d\n", e, whileEnd);
  skip(")");
  STMT();
  emit("goto L%d\n", whileBegin);
  emit("(L%d)\n", whileEnd);
}

void STMT() {
  if (isNext("while"))
    return WHILE();
  else if (isNext("do"))
    DOWHILE();
  else if (isNext("if"))  //
     IF();                //
  else if (isNext("{"))
    BLOCK();
  else
    ASSIGN();
}

void STMTS() {
  while (!isEnd() && !isNext("}")) {
    STMT();
  }
}

// { STMT* }
void BLOCK() {
  skip("{");
  STMTS();
  skip("}");
}

void PROG() {
  STMTS();
}

void parse() {
  printf("============ parse =============\n");
  tokenIdx = 0;
  PROG();
}
```
# Output
```
>compiler test/dowhile.c
s=0;
i=1;
do {
  s = s + i;
  i = i + 1;
} while (i <= 10);
========== lex ==============
token=s
token==
token=0
token=;
token=i
token==
token=1
token=;
token=do
token={
token=s
token==
token=s
token=+
token=i
token=;
token=i
token==
token=i
token=+
token=1
token=;
token=}
token=while
token=(
token=i
token=<
token==
token=10
token=)
token=;
========== dump ==============
0:s
1:=
2:0
3:;
4:i
5:=
6:1
7:;
8:do
9:{
10:s
11:=
12:s
13:+
14:i
15:;
16:i
17:=
18:i
19:+
20:1
21:;
22:}
23:while
24:(
25:i
26:<
27:=
28:10
29:)
30:;
============ parse =============
t0 = 0
s = t0
t1 = 1
i = t1
(L0)
t2 = s
t3 = i
t4 = t2 + t3
s = t4
t5 = i
t6 = 1
t7 = t5 + t6
i = t7
t8 = i
t9 = 10
t10 = t8 <= t9
if T10 goto L0
```