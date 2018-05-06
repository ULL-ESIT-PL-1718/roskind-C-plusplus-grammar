Copyright (C) 1989.1990, James  A.   Roskind,  All  rights  reserved. 
Abstracting  with credit is permitted.  The contents of this file may 
be reproduced electronically, or in printed  form,  IN  ITS  ENTIRETY 
with  no  changes, providing that this copyright notice is intact and 
applicable in the copy.  No other copying is  permitted  without  the 
expressed written consent of the author.

FILENAME: GRAMMAR5.TXT
AUTHOR: Jim Roskind
        Independent Consultant
        516 Latania Palm Drive
        Indialantic FL 32903
        (407)729-4348
        jar@hq.ileaf.com
        or ...!uunet!leafusa!jar


    A YACC-able C++ 2.1 GRAMMAR, AND THE RESULTING AMBIGUITIES
                                        (Release 2.0 Updated 7/11/91)

ABSTRACT

This paper describes ambiguous aspects of the C++ language  that  have 
been  exposed by the construction of a YACC-able grammar for C++.  The 
grammar is provided in  a  separate  file,  but  there  are  extensive 
references  to  the  actual  grammar.   This  release  of  the grammar 
includes the output from a  yacc  cross-reference  tool  that  I  have 
written.   The  discussion  in  this file will make many references to 
that verbose output (typically provided in a  file  called  y.output), 
but  the  file  is only critical if you are trying to "check my work", 
rather than read my results.  The  format  of  the  machine  generated 
documentation provided in y.output is defined in autodoc5.txt.

This  release  of  the  grammar  provides  what  I believe is complete 
support for nested types, at least syntactically.  It does not support 
either templates, or exception handling,  as  the  discussion  of  the 
syntax for these elements is not finalized.  

Theoretical  Note  (skip  if you hate theory): My C++ grammar does not 
make use of the %prec or %assoc features of YACC, and hence  conflicts 
are  never  hidden within my grammar.  The wondrous result is that, to 
the extent to which my grammar can be seen to span the syntax  of  the 
language,  in  all places where YACC reports no conflicts, the grammar 
is a totally unambiguous statement of  the  syntax  of  the  language.  
This  result  is  remarkable  (I think), because the "general case" of 
deciding whether a grammar is ambiguous or not, is equivalent to  (the 
unsolvable) halting problem.

Although  this  paper is terse, and at least hastily formed, I believe 
that its content is so significant to my results that  it  had  to  be 
included.   I  am  sorry  that I do not have the time at this point to 
better arrange its contents.




CONTENTS

	CONTENTS
        INTRODUCTION
        REVIEW: STANDARD LEXICAL ANALYSIS HACK: TYPEDEFname vs IDENTIFIER
        STATUS OF MY "DISAMBIGUATED" GRAMMAR
	SUMMARY OF CONFLICTS
        
	17 EASY CONFLICTS, WITH HAPPY ENDINGS
	1 SR CONFLICT WITH AN ALMOST HAPPY ENDING
        6 NOVEL CONFLICT THAT YIELD TO SEMANTIC DISAMBIGUATION
	1 CONFLICT THAT CONSTRAINTS SUPPORT THE RESOLUTION FOR
	
	THE TOUGH AMBIGUITIES: FUNCTION LIKE CASTS AND COMPANY (17 CONFLICTS)
        THE TOUGH AMBIGUITIES: FUNCTION LIKE CASTS AND COMPANY
	LALR-only CONFLICTS IN THE GRAMMAR
        SAMPLE RESOLUTIONS OF AMBIGUITIES BY MY C++ GRAMMAR
        DIFFICULT AMBIGUITIES FOR C++ 2.0 PARSER TO TRY
        COMMENTARY ON CURRENT C++ DISAMBIGUATING RULES
        SOURCE OF CONFLICTS IN C++ (MIXING TYPES AND EXPRESSIONS)
	FUNCTION LIKE CAST vs DECLARATION AMBIGUITIES
	FUNCTION LIKE CAST vs DECLARATION : THE TOUGH EXAMPLE:        
        
	CONCLUSION

	APPENDIX A:
	  PROPOSED GRAMMAR MODIFICATIONS (fixing '*', and '&' conflicts)

	APPENDIX B:
	  CANONICAL DESCRIPTION OF CONFLICTS and STATES





INTRODUCTION

This paper is  intended  to  go  into  significant  detail  about  the 
ambiguities  that I have seen in the C++ 2.1 language (proposed in the 
"C++ Annotated Reference Manual", a.k.a. ARM, by Ellis and Stroustrup) 
and exposed by attempting to develop a YACC compatible (i.e., LALR(1)) 
grammar.  I must point out that this is NOT meant as an attack on  the 
language  or  the  originators  thereof,  but  rather an R&D effort to 
disambiguate some details.  (I personally believe that Stroustrup  et. 
al.  are doing a GREAT job).  I have the vague hope that the extensive 
hacking  that  I have done with YACC on the C++ grammar has given me a 
rather novel vantage point.  (I  have  expressed  my  observations  to 
Bjarne  Stroustrup,  and in doing so verified that other folks had not 
previously  identified  several  of  my  points).   In  light  of   my 
activities,  this  paper  includes  a fair amount of philosophizing. I 
hope that none of this paper assumes too greatly that I  have  insight 
that  is  beyond  that  of  the  readers, and certainly no insults are 
intended.

If you like investigating grammars directly (rather than reading  what 
I  have  to say), I would strongly suggest you read the ANSI C grammar 
before looking at the C++  grammar.   Additionally,  if  you  want  to 
really  understand  the  grammar,  as  suggested  in autodoc5.txt, the 
grammar cross-reference provided in y.output is also very helpful. The 
C++ grammar  is  pretty  large,  and  you  can  expect  the  following 
statistics if you have a similar YACC to that I am using:

Berkeley YACC (1.8  01/02/91)

103 terminals, 186 nonterminals
665 grammar rules, 1256 states
24 shift/reduce conflicts, 18 reduce/reduce conflicts.

Sadly,  many  standard  yacc implementations have static limits on the 
grammars that they can handle.  If you  find  this  to  be  a  problem 
(i.e.,  you  yacc  says  "too  many  states", and terminates), I would 
suggest acquiring Berkeley yacc (which is free), or  bison  (which  is 
mostly  free),  or  purchasing  a  commercial  yacc implementation.  A 
simple port of these publicly available sources on  machines  with  16 
bit  "int"s  (e.g.,  most  PCs)  will  also  be  unable to process the 
grammar.  PCYACC (from Abraxas  Software)  and  ydb  (from  Bloomsbury 
Software),  are examples of commercial products that run on the PC and 
Sun respectively, that I know from experience can yacc these  grammars 
(I currently have no financial affiliation with either vendor).


REVIEW: STANDARD LEXICAL ANALYSIS HACK: TYPEDEFname vs IDENTIFIER

This section is targeted at readers that are not familiar with parsing 
C  with a context free grammar.  The reader should note that there are 
two distinct forms of `identifiers' gathered during lexical  analysis, 
and  identified  as  terminal  tokens  in  both  of  my grammars.  The 
terminal tokens are called IDENTIFIER  and  TYPEDEFname  respectively. 
This  distinction  is  a  required  by  a fundamental element of the C 
language.  The definition of a "TYPEDEFname" is a  lexeme  that  looks 
like  a  standard  identifier,  but is currently defined in the symbol 
table as being declared a typedef (or class, struct, enum  tag)  name. 
All  other  lexemes  that appear as identifiers (and are not keywords) 
are tokenized as IDENTIFIERs.  The  remainder  of  this  section  will 
review the motivation for this approach.

Consider the following sample code, which uses the C language:

        ...
        int main() { f (*a) [4] ; ...

Most  readers  will intuitively parse the above sequence as a function 
call to "f", with an argument "*a",  the  return  value  of  which  is 
(probably)  a  pointer,  and  hence  can  be  indexed by "4".  Such an 
interpretation presumes that the prior context was something like:

        ...
        extern float *a;
        char * f(float);
        int main() { f (*a) [4] ; ...

However, if the prior context was **INSTEAD**:

        ...
        typedef int f;
        extern float a;
        int main() { f (*a) [4] ; ...

then the interpretation  of  the  code  is  entirely  different.   The 
fragment  in  question  actually redeclares a local variable "a", such 
that the type of "a" is "pointer to array of 4 int's"!  So we  see  in 
this   example   that  the  statement  "f(*a)[4];"  cannot  be  parsed 
independent of context.

The standard solution to the above ambiguity is to allow  the  lexical 
analyzer to form different tokens based on contextual information. The 
contextual information that is used is the answer to the question: "Is 
a  given  identifier  defined as a typedef name (or class name) at the 
current point in the parse"?  I will refer to this feedback loop (from 
the parser that stores information in  a  symbol  table,  wherein  the 
lexer extracts the information) as the "lex hack".  With this lex hack 
in  place the code fragment "f(*a)[4];" would be provided by the lexer 
as either:

        IDENTIFIER '(' '*' IDENTIFIER ')' '[' INTEGERconstant ']' ';'

or

        TYPEDEFname '(' '*' IDENTIFIER ')' '[' INTEGERconstant ']' ';'

The two case are very easy for a context free grammar to  distinguish, 
and  the  ambiguity  vanishes.  Note that the fact that such a hack is 
used (out of necessity) demonstrates that C  is  not  a  context  free 
language,  but  the hack allows us to continue to use an LR(1) parser, 
and a context free grammar.

Note that this hack is, of necessity, also made  use  of  in  the  C++ 
grammar,  but  no  additional  feedback  (a.k.a.:  hack)  is (at least 
currently) required.  The interested reader should also note that this 
feedback loop (re: updating the symbol table) must be  swift,  as  the 
lexical  analyzer  cannot  proceed  very  far  without this contextual 
information.  This constraint on the feedback time often prevents  the 
parser  from  "deferring" actions, and hence increases the pressure on 
the parser to rapidly disambiguate token sequences.



STATUS OF MY "DISAMBIGUATED" GRAMMAR

Several independent reviews,  which  provided  a  complete  front  end 
lexical  analyzers,  and  parsed existing C++ code, have verified that 
the grammars span of  the  C++  language  appears  complete.   I  have 
neither   incorporated   parametric   types  (a.k.a.   templates)  nor 
exception handling into the grammars at this point, as  they  continue 
to be in a state of flux.

The  grammar does (I believe) support all the features provided in C++ 
2.1, including multiple inheritance and the  enhanced  operator  "new" 
syntax (includes placement expression). I believe that, except for the 
minor  change  involving  not  permitting parentheses around bit field 
names during a declaration, my C++ grammar supports a superset  of  my 
ANSI  C  grammar. Note that I haven't inline expanded all the rules in 
the C grammar that were required for C++ disambiguation (re: deferring 
reductions), and hence a `diff' of the two grammars will not provide a 
trivial comparison.  The resulting major  advantage  of  this  grammar 
over  every  other  current  C++  parser  (that  I know of) is that it 
supports old style function definitions AS WELL AS  all  the  standard 
C++.   (It is my personal belief that such support was dropped by many 
compilers and translators in order to resolve the many syntax problems 
that appear within C++.  I believe this  grammar  shows  that  such  a 
decision was premature).

My  list  of shift-reduce and reduce-reduce conflicts is currently: 24 
shift/reduce, 18 reduce/reduce conflicts reported.  I have  chosen  to 
leave  so  many conflicts in the grammar because I hope to see changes 
to the syntax that will remove them, rather than making changes to  my 
grammar  that  will firmly accept and disambiguate them.  (Considering 
the detailed analysis presented here,  such  changes  would  only  add 
unnecessary complications to an already large grammar).


SUMMARY OF CONFLICTS

The  following  summarizes the conflicts based on a simple description 
of the nature of the conflict:

8 SR caused by operator function name with trailing * or &
	states: 131, 138, 281, 282

8 SR caused by freestore              with trailing * or &
	states: 571, 572, 778, 779

1 SR caused by dangling else and my laziness
	state: 1100

1 SR caused by member declaration of sub-structure, with trailing :
	state: 64

6 RR caused by constructor declaration vs member declaration
	states: 1105, 1152, 1175

1 SR caused by explicit call to destructor, without explicit scope
	state: 536




3 RR caused by function-like cast vs typedef redeclaration ambiguity
	state: 738

3 RR caused by function-like cast vs identifier declaration ambiguity
	state: 739

3 RR caused by destructor declaration vs destructor call 
	state: 740

3 RR caused by parened initializer vs  prototype/typename
	state: 758

5 SR caused by redundant parened TYPEDEFname redeclaration vs old style cast
	states: 395, 621, 1038, 1102, 1103



Of these conflicts, the ones that most C++ parser authors  are  mainly 
concerned  with  the  17  listed  at  the end of the above list.  They 
relate to function-like-cast vs  declaration,  and  redundant  parened 
TYPEDEFname  redeclaration  vs  old  style  cast,  etc.  The following 
sections breeze through the "easy" conflicts, and then talk at  length 
about these tough ones.



17 EASY CONFLICTS, WITH HAPPY ENDINGS

The first group of 17 SR conflicts:

8 SR caused by operator function name with trailing * or &
	states: 131, 138, 281, 282
8 SR caused by freestore              with trailing * or &
	states: 571, 572, 778, 779
1 SR caused by dangling else and my laziness
	state: 1100

have  very simple resolutions.  If you are reading this, I assume that 
you are already familiar with the  if-if-else  conflict  (it  is  also 
analyzed in depth in the autodoc5.txt file).

The  8  conflicts based "freestore with trailing * or &" can be hinted 
at by the example:

        a = new int * * object;

Is the above the same as:

        a = (new int) * (* T);
or:
        a = (new (int *)) * T;

As a resolution,  the  "longest  possible  type"  is  isolated  by  my 
grammar.  The result is:

        a = (new (int * * )) ...

which  leads  to  a  syntax  error  in my example!  This resolution is 
indeed what is quietly specified  for  C++  2.0  (and  implemented  in 
cfront).  The critical statement and example in the C++ 2.0 Ref Man at 
the end of section 5.3.3 makes this resolution clear.

The  8 conflicts involving "operator function names with trailing * or 
&" are quite similar to what was just presented.  The critical fact is 
that "operator typename"  is  allowed  in  the  grammar  to  define  a 
function.  Whenever a function is provided, but NOT followed by a '(', 
the  address  of  the  function  is  implicitly  taken and used in the 
expression (see draft ANSI C standard for finer  details).   For  some 
class T, the following MIGHT all be valid statements:

        operator T;
        operator T*;
        operator T**;

If  the  above  are valid, then the interpretation of the following is 
ambiguous:

        operator T * * a;

The above might be interpreted as:

        (operator T) * (* a);
or
        (operator (T *)) * a;

The default LR rules parse the largest possible type, and lead to:

        (operator (T * * )) ...

which in our example leads to a syntax error.  Here again the "longest 
possible type..." rule supports my grammar.  Note that  this  rule  is 
actually  a  consequence  (in my opinion) of the cfront implementation 
via a YACC grammar, and the default resolution of  conflicts  in  that 
grammar.



1 SR CONFLICT WITH AN ALMOST HAPPY ENDING

1 SR caused by member declaration of sub-structure, with trailing :
	state: 64

Note that this conflict is different from the one isolated in the 6/90 
version of this grammar, that pertained to ':'.

The conflict now takes place (as seen in the demonstrations section of 
y.output) in variations of the following program prefix:

	class A { class B :

The  problem  is  that  "class B" can be a declaration specifier, much 
like "int".  When a bit field is defined, then  "int  :"  provides  an 
unnamed  bit  field  member  of  a  structure.   My grammar avoids the 
reduction of "class B" to declaration specifier in this  context,  and 
hence  disambiguates  in  favor  of  a nested class, with a derivation 
list:

	class A { class B : Parent { ..... };};

Although this  looks  reasonable  today,  when  template  classes  are 
introduced,  and  the  supplied  type may be "int", then a real syntax 
problem  will  be  present   (hence,   the   almost   happy   ending). 
Specifically,  one  can  image a parameterized type, which is given as 
input (i.e., an argument) either "signed int", or "unsigned int",  and 
then  this  type  might  be  used  as a declaration specifier in a bit 
field.  When the type is referred to during  the  parameterized  class 
elaboration,  the  reference *can* be made in the form "class T", even 
though "T" is simply "signed int".

For now (until the template syntax is worked out), the  disambiguation 
provided by this grammar will do very nicely.



6 NOVEL CONFLICTS THAT YIELD TO SEMANTIC DISAMBIGUATION

The  conflicts  that  are discussed in this section have been deferred 
(by A LOT of work, and A LOT of inline expansion) to occur when a ';', 
or a '{' is reached.  At  that  point,  semantic  information  in  the 
tokens can safely be used to decide which of two cases are at hand.

The conflicts are referred to as:

    6 RR caused by constructor declaration vs member declaration
	states: 1105, 1152, 1175


occur during a class/struct elaboration.  Consider the following class 
elaborations:

        typedef int T1, T2, T3  ;
        class GOO { int a;} ;
        class FOO {
                GOO    T1  ; // clearly a redefinition of T1
                FOO  ( T2 ); // clearly a constructor
                GOO  ( T3 ); // redefinition of T3
                };

Note  that  the  last  two entries in FOO's elaboration "FOO(T2);" and 
"GOO(T3);" are  tokenized  IDENTICALLY,  but  must  have  dramatically 
different  meanings.   When I first found this ambiguity I was hopeful 
that I could extend the lex hack that distinguishes TYPEDEFnames  from 
random  IDENTIFIERs,  and distinguish something like CURRENTCLASSname. 
Unfortunately, the  potential  for  elaborations  within  elaborations 
appears  to  make  such a hack unworkable.  In addition, once I got my 
grammar to defer all such ambiguous cases until a ';' was seen, I felt 
confident that the ambiguity was resolved (and the introduction of  an 
additional "hack" was unnecessary).

Note  that  the  situations  are  identical  when a '{' is seen, as it 
presents the start of the body of either a function, or a constructor, 
and an identical decision must be made.


1 CONFLICT THAT CONSTRAINTS SUPPORT THE RESOLUTION FOR

With  this  new  grammar,  the  ability  to  make  explicit  calls  to 
constructors is supported.  As pointed out in section 12.4 of the ARM, 
the  implicit  use  of  the  "this"  pointer  to  refer  to a specific 
destructor leads to an ambiguity with the unary "~" operation.   As  a 
result, in situations where it is possible to parse a sentence so that 
the "~" is the unary operator, it is done.  The conflict is shown as:

1 SR caused by explicit call to destructor, without explicit scope
	state: 536

Note that the reduction:

	complex_name : '~' TYPEDEFname 

is  what  is  used  to develop a "name" that can be used to refer to a 
destructor  explicitly.   The  decision  is  made  to  not  use   this 
reduction,  in favor of "something else" (which results from a shift). 
Since the only alternative to specifying a destructor is to  make  "~" 
serve as a unary operator, we are assured that we support the standard 
given in the ARM.  Note that this should probably be officially listed 
as  a part of the syntax restrictions, but in any case, it is at least 
a disambiguating constraint, and we are guaranteed to support it.



THE TOUGH AMBIGUITIES: FUNCTION LIKE CASTS AND COMPANY (17 CONFLICTS)

The  ambiguities  listed  in  this  section  pertain  to  attempts  to 
distinguish                declaration/types-names                from 
expression-statements/expressions.  For example:

    char *b ="external" ; // declare a variable to confuse us :-)
    main () {
        class S;
        S (*b)[5]; // redeclare "b" pointer to array of 5 S's ?
               // OR ELSE indirect through b; cast to S; index using 5 ?
        }

The above is what I  call  the  "declaration  vs  function  like  cast 
ambiguity".   Awareness  about  this ambiguity in this context appears 
fairly widespread among C++ parser authors.  The  ARM  makes  explicit 
reference  to  this  problem in section 6.8 "Ambiguity Resolution".  I 
believe the underlying philosophy provided by the Reference Manual  is 
that  if a token stream can be interpreted by an ANSI C compiler to be 
a declaration, then a C++ compiler should disambiguate in favor  of  a 
declaration.  Unfortunately, section 6.8 goes on to say:

    "To  disambiguate,  the whole statement may have to be examined to 
    determine if it is an expression-statement, or a declaration.  ... 
    The disambiguation is purely syntactic; that is,  the  meaning  of 
    the  names, beyond whether they are type-names or not, is not used 
    in the disambiguation".

The  above  advice  only  forestalls  the  inevitable  ambiguity,  and 
complicates  the  language  in  the process.  The examples that follow 
will demonstrate the difficulties.

There are several other contexts where such  ambiguities  (typedef  vs 
expression) arise:

        1) Where a statement is valid (as shown above).
        2) As the argument to sizeof()
        3) Following "new", with the C++ syntax allowing a placement
                expression
        4) Immediately following a left paren  in  an  expression  (it 
                might be an old style cast, and hence a type name)
        5) Following  a  left paren, arguments to constructors can be 
                confused with prototype type-names.
        6) Recursively in any of the above,  following  a  left  paren 
                (what  follows might be argument expressions, or might 
                be function prototype parameter typing)

Examples of simple versions of the sizeof context are:

        class T;
        sizeof ( T    ); // sizeof (type-name)
        sizeof ( T[5] ); // again a type name
        sizeof ( T(5) ); // sizeof (expression)
        sizeof ( T()  ); // semantic error: sizeof "function returning T"?
                        // OR ELSE sizeof result of function like cast

Examples  of  the  old  style  cast ambiguity context, which are still 
ambiguous when the '(' after the 'T' has been seen:

        class T { /* put required declarations here */ 
                };
        a = (T(      5));  // function like cast of 5
        b = (T(      )) 0; // semantic error: cast of 0 to type "function
                        // returning T"

In constructors the following demonstrates the problems:

        class T;
        T (b)(int  d ); // same as "T b(int);", a function declaration
        T (d)(int (5)); // same as "T d(5);", an identifier declaration
        T (d)(int ( )); // ambiguous

The problem can appear recursively  in  the  following  examples.   By 
"recursively"  I  mean  that an ambiguity in the left-context has made 
the parser unsure of whether an "expression"  or  a  "type"  is  being 
parsed,  and  the ambiguity is continued by the token sequence.  After 
the parser can determine what this subsequence is, it will in turn  be 
able to disambiguating what the prior tokens were.

Recursion on the statement/declaration context:

        class S;
        class T;
        S (*b)(T); // declare b "pointer to function taking T returning S"
        S (*c)(T dummy); // same declaration as for "b"
        int dummy;
        S (*d)(T (dummy)); // This T might be casting dummy

Recursion on the sizeof context is shown in the following examples. As 
before, the examples include semantic errors.

        class T;
        class S;
        sizeof ( T(S dummy) ); // sizeof "function taking S returning T"
        int dummy;
        sizeof ( T(S (dummy)) ); // sizeof "function taking S returning T"
                // OR ELSE cast dummy to S, and then cast that to T, which
                        // is the same as "sizeof T;"


The  following are the list of conflicts that fall into the categories 
listed above.  To see the complete details of each conflict state  see 
the y.output file supplied with this grammar.


3 RR caused by function-like cast vs typedef redeclaration ambiguity
	state: 738

3 RR caused by function-like cast vs identifier declaration ambiguity
	state: 739

3 RR caused by destructor declaration vs destructor call 
	state: 740

3 RR caused by parened initializer vs  prototype/typename
	state: 758

5 SR caused by redundant parened TYPEDEFname redeclaration vs old style cast
	states: 395, 621, 1038, 1102, 1103

In  each  case  the  conflicts are resolved in favor of a declaration.  
Note  that  this  decision  is  made  when  the  "declarator"  *seems* 
complete.  Typically this takes place when a character (such as a ')', 
',', or '=') that terminates a declarator or abstractor declarator  is 
seen.  At  such  a  point,  an  my LR(1) grammar makes the decision to 
support a declaration/type-name assuming it is possible.   (The  added 
complexity of the LALR-only conflicts discussed in the next section). 

Note  that  this disambiguation (provided by my grammar) is *NOT* what 
is suggested in the ARM.   The  ARM  suggests  what  some  folks  have 
referred  to  as "advanced parsing techniques" be used to disambiguate 
based on first "pre-parsing" arbitrarily far ahead.  This approach  is 
actually  not  "advanced  parsing", rather it is a through back to the 
days before parsing was understood, and exponential search  algorithms 
were  proposed  as  "parsing techniques." Currently, cfront supports a 
hacked partial look-ahead technique, that alternately  "disambiguates" 
hard  examples  and  "core-dumps"  for  others.  To date, I know of no 
implementation  that  actually   comes   near   matching   the   ARM's 
specification,   and   I  suspect  that  attempts  to  implement  such 
conformance will lead to many buggy compilers,  that  support  varying 
degrees  of  compliance  with  such ad-hoc requests for parsing.  Only 
time will tell whether  the  ANSI  C++  committee  will  support  this 
unimplemented  disambiguation technique.  Most users of cfront *think* 
that the ARM is implemented in cfront, and that cfront implements  the 
ARM.   This  area  of ambiguity not only demonstrates cfront's lack of 
compliance  with  the  ARM,  but  also   exemplifies   some   of   the 
unimplemented technology that is proposed in the ARM.


LALR-only CONFLICTS IN THE GRAMMAR

As  can  be seen in the analysis of the grammar in y.output, there are 
some  LALR-only  conflict  contexts  in  some  of  the   reduce/reduce 
conflicts.   Since yacc disambiguates in favor of lower numbered rules 
during  R/R  conflict,  most  of   these   LALR-only   conflicts   are 
insignificant  (see  autodoc5.txt  for  discussion  of  this concept). 
Specifically, when the left context *requires* unambiguously that that 
a certain reduction take place, and the parser was going  to  do  that 
reduction   anyway   (by  default),  then  the  LALR-only  problem  is 
insignificant.  Based on the above reasoning, the LALR-only  conflicts 
in the following states are insignificant:

	states: 738(mostly), 739, 740, 1105

The  states where there are real LALR-only conflicts with significance 
are:
	
	state 738 on ',' and '='

	state 758 on ',' and '='

The significant subsets of the unambiguous left context trees  (copied 
from the y.output file) are:


LALR-only conflict contexts leading to state 738 and lookahead symbol <','>
	Possible reductions rules include (61,73)
		type_qualifier_list_opt : (61)
		postfix_expression : TYPEDEFname '(' ')' (73)

--738+-1014--952+-837(73)    $start IDENTIFIER '{' ! CLCL TYPEDEFname '(' TYPEDEFname '(' ')'
     |          --835(73)    $start IDENTIFIER '{' TYPEDEFname ! '(' TYPEDEFname '(' ')'
     |
     --748(73)    $start IDENTIFIER '(' '(' TYPEDEFname ! '(' ')'


LALR-only conflict contexts leading to state 738 and lookahead symbol <'='>
	Possible reductions rules include (61,73)
		type_qualifier_list_opt : (61)
		postfix_expression : TYPEDEFname '(' ')' (73)

--738+-1014--952(73)    $start IDENTIFIER '{' TYPEDEFname '(' ! TYPEDEFname '(' ')'
     |
     --748(73)    $start IDENTIFIER '(' '(' TYPEDEFname ! '(' ')'


LALR-only conflict contexts leading to state 758 and lookahead symbol <','>
	Possible reductions rules include (61,74)
		type_qualifier_list_opt : (61)
		postfix_expression : global_or_scoped_typedefname '(' ')' (74)

--758--754(74)    $start IDENTIFIER '(' '(' ! CLCL TYPEDEFname '(' ')'


LALR-only conflict contexts leading to state 758 and lookahead symbol <'='>
	Possible reductions rules include (61,74)
		type_qualifier_list_opt : (61)
		postfix_expression : global_or_scoped_typedefname '(' ')' (74)

--758--754(74)    $start IDENTIFIER '(' '(' ! CLCL TYPEDEFname '(' ')'


As explained in the description of the "sample sentence" that is given 
for  each  context  (see  autodoc5.txt),  it  is not unique, and it is 
*especially* non-unique to the left of  the  '!'  symbol.   Never  the 
less, it provides (most of the time) rapid insight into the conflicts, 
which  makes  it  unnecessary to look at the details of the associated 
demonstrations (also provided in the y.output file).

Looking at these examples, it  can  be  seen  that  in  each  case,  a 
parenthesized  type  name  is  created,  and  a trailing ',' or '=' is 
found.  Since the type-name used in an old style cast  cannot  contain 
such  symbols  as  ',' or '=' (unprotected by parens), the presence of 
such a token  precludes  the  possibility  of  the  sequence  being  a 
typename.  For example, looking at the left context for state 738 that 
includes state 952, consider the fragment:

	struct  TYPE1 { ...... } ;
	struct  TYPE2 { ...... } ;
	main(){
		TYPE1 ( ( TYPE2()


There  is  a chance that TYPE2 is about to be redeclared at this inner 
scope.  In such a case the fragment might continue:


	struct  TYPE1 { ...... } ;
	struct  TYPE2 { ...... } ;
	main(){
		TYPE1 ( ( TYPE2()));
		}

with lots of redundant parents around the redeclared "TYPE2".   As  an 
alternative, the fragment might continue:

	struct  TYPE1 { ...... } ;
	struct  TYPE2 { ...... } ;
	main(){
		TYPE1 ( ( TYPE2() + 10));

and  become an expression.  The problematic situation is provided when 
(for example) a ',' follows:

	struct  TYPE1 { ...... } ;
	struct  TYPE2 { ...... } ;
	main(){
		TYPE1 ( ( TYPE2() , 

Clearly, as provided in the analysis, this must be a case  where  only 
an  expression  can  result,  as a ',' could not be placed as shown if 
TYPE2 were being redeclared.

Careful consideration of each of the other LALR-only conflicts reveals 
an identical fundamental problem.

As is the basis for all  LALR-only  contexts,  there  are  some  other 
contexts  where this same LALR states 738 is used with a trailing ',', 
and it is possible for the item to  the  left  of  the  ','  to  be  a 
declarator  (specifically,  when the declarator is the first type in a 
prototype list, a comma may follow it).  Interestingly,  this  precise 
problem  is also the reason for the '=' look-ahead problem, as '=' may 
be used to specify an initializer in a prototype list, but  can  never 
be  found  in  an abstract declarator (as is required for an old style 
cast typename), or  adjacent  to  a  simple  declaration's  declarator 
*while* nested within parenthesis.

With  this  insight, it is not difficult to change the grammar so that 
the critical rules are repeated for these two distinct contexts.  As I 
mentioned in my autodoc5.txt file,  I  am  working  on  a  subtle  and 
automated  method  for  removing  these  LALR-only conflicts.  I would 
prefer (for now) to leave these conflicts in place and remove them via 
subtle  means,  rather  than  by  brute   replications   of   critical 
reductions.   Since  the nesting complexities of these scenarios seems 
large (re: function like cast of function like cast  etc.),  I  expect 
that  leaving  these  conflicts  in  the  grammar will have no adverse 
effects on users with real code.   Recall  that  the  worst  that  can 
happen  with  such  conflicts  remaining  is that an expression can be 
signaled as a syntax error, when in fact it is "unambiguous", based on 
one token of look-ahead.


SAMPLE RESOLUTIONS OF AMBIGUITIES BY MY C++ GRAMMAR

Since my grammar tends to disambiguate "prematurely" (i.e., the parser 
does not use infinite lookahead) when compared with the ARM  standard, 
it tends to force non-declarations into being declarations.  Later on, 
when   the   tokens  appear  that  require  an  interpretation  as  an 
expression, a syntax error results.  Hence, the  "hard  examples"  are 
those   in   which  the  disambiguating  tokens  appear  late  in  the 
expressions.

Of the "hard examples" given in the ARM (r6.8), my  grammar  can  only 
"properly" detect a "statement-expression" for the stream:

        T(a,5)>>c;

All  the  other  examples  default  to  a declarator after the closing 
parenthesis  following  the  identifier.   (See  my  comments  in  the 
conclusion section of this paper).

I  actually  am  not sure I agree with all the examples in the C++ 2.0 
Reference Manual. Specifically, the example in section 6.8:

        T (*d) (double(3));  // expression statement

In the example "T"  is  specified  to  be  a  simple-type-name,  which 
includes  all  the  basic  types  as  well  as  class-names, and more. 
Considering the following are valid declarations:

        void *a (0);
        void *b (int(0));
        void (*c)(int(0));

I am unable to see the "syntactic" difference between this last  token 
stream  and  the  example  just  cited  in  the  reference manual.  My 
simplistic parser gives me the result that  I  at  least  expect.   It 
concludes  (prematurely, but seemingly correctly) that the stream is a 
declaration (with a new style initializer).

As a positive note, my grammar is able to parse the  example  given  a 
while back in comp.lang.c++, that Zortech C++ 1.07 could not parse:

        a = (double(a)/double(b))...;

Apparently,   upon   seeing   "(double"   some  parsers  commit  to  a 
parenthesized type-name for a cast expression, and cannot  proceed  to 
parse  a  parenthesized  expression.   No  mention  of this problem is 
listed in my conflict list, as resolution of this problem is simply  a 
matter of letting the LR parser wait long enough before committing. My 
grammar commits when it sees "a" in "(a)" to providing an expression.


DIFFICULT AMBIGUITIES FOR A "C++ 2.0" COMPATIBLE PARSER TO TRY

Having seen the above contexts, I would be curious to see if other C++ 
front  ends  with  "smart  lexers"  (such  as  cfront)  can handle the 
following.   These  examples  are  not  guaranteed  to  be   evaluated 
correctly  by  my grammar, but I expect them to demonstrate weaknesses 
in many other parsers.  The interpretation of these examples  per  C++ 
2.0 definitions requires massive lookahead.  In addition, the examples 
are  generally unreadable by humans, and rarely parsed the same way by 
any two implementations.

    main()
      {
      class T 
        { 
        /* ... */
        } a;
      typedef T T1,T2,T3,T4,T5,T7,T8,T9,Ta,Tb,Tc,Td;
        { /* start inner scope */
        T((T1)         ); // declaration
        T((T2)    a    ); // Statement expression
        T((T3)(       )); // declaration of T3
        T((T4)(T      )); // declaration of T4
        T((T5)(T  a   )); // declaration of T5
        T((T6)(T((a) ))); // declaration of T6
        T((T7)(T((T) ))); // declaration of T7
        T((T8)(T((T)b))); // statement expression

        T(b[5]); // declaration
        T(c());  // declaration
        T(d()[5]);  // statement expression ? (function returning array 
                      // is semantically illegal, but syntactically proper)
        T(e[5]());  // statement expression ? (No array of functions)
        T(f)[5]();  // statement expression ?  "                   "
        
        T(*Ta() )[5] [4];  //declaration
        T(*Tb() [5]) [4];  //statement expression ? (function returning array)

        T(*Tc()) [3 +2];  //declaration
        T(*Td()) [3 ]+2;  //statement expression

        }
      }        


COMMENTARY ON C++ 2.0 DISAMBIGUATING RULES

There are two distinct thrusts in conflict disambiguation as  provided 
by  AT&T's  efforts to define a standard for C++.  The first thrust is 
"parse tokens into the longest possible declarator, and  identify  the 
syntax  errors  that  result".   The  second thrust is to "use massive 
external technology ("smart lexer", a.k.a.: "recursive  decent  parser 
that  helps  the  lexer",  a.k.a.   LALEX)  to look ahead, so that the 
parser doesn't mis-parse a function-like-cast  as  a  declaration  and 
induce  a  syntax  error".   The  first  is  a commitment to LR parser 
technology, and an existing grammar (which could be cleaned up).   The 
second  is  a commitment to NOT use an LR parser, and to the use of an 
existing implementation.

It is my belief that LR parsers are well understood, and the  addition 
of a "smart lexer" destroys all structure in a parser.  The result can 
be anticipated to become a quagmire of code and hacks.  With this firm 
conviction,  I  have  provided my grammar in the hopes that a standard 
can emerge that IS well defined, and is implementable, and is readable 
by humans.


SOURCE OF CONFLICTS IN C++ (MIXING TYPES AND EXPRESSIONS)

One fundamental strength in C is the similarity  between  declarations 
and  expressions.   The  syntax  of  the  two  is  intended to be very 
similar, and the result is a clean declaration and expression  syntax. 
(It  takes  some  getting  used  to,  but  it  is in my opinion good).  
Unfortunately, there are some slight distinctions  between  types  and 
expressions,  which  Ritchie  et.  al.  apparently noticed.  It is for 
this reason (I am guessing) that the C cast operator REQUIRES that the 
type be enclosed in parenthesis.  Moreover,  there  is  also  a  clear 
separator in a declaration between the declarator and the initializing 
expression  (the  '=') (as some of you know, there is some interesting 
history  in  this  area.).   The  bottom  line  (as  seen  with  20-20 
hindsight)  is:  "keep  declarations  and expressions separate".  Each 
violation of this basic rule has induced conflicts.


To be concrete about the differences between  types  and  expressions, 
the following two distinctions are apparent:

    1)  Abstract declarators are permitted.  No analogy is provided in 
    expressions.  The notable distinction is that abstract declarators 
    include the possibility of trailing '*' tokens.
        
    2) The binding of elements in a declaration is very different from 
    any expression.  Notably,  the  declaration-specifiers  are  bound 
    separately  to  each  declarator  in  the  comma separated list of 
    declarators (example:  int  a,  b,  c;).   With  (most  forms  of) 
    expressions,   a   comma   provides   a  major  isolation  between 
    expressions.

C also used  reserved  names  to  GREATLY  reduce  the  complexity  of 
parsing.   The  introduction of typedef names increased the complexity 
(it made the language context sensitive), but a  simple  hack  between 
lex and YACC overcame the problem.  An example is the statement:

        name (*b)[4];

Note  that  this  is  ambiguous,  EVEN  in  ANSI  C,  IF  there  is no 
distinction between type-names and function names!  (i.e.,  "b"  could 
be getting redeclared to be of type "pointer to array of name", OR the 
function  "name"  could  be  called  with argument "*b", the result of 
which is indexed to the 4th element). In C, the  two  kinds  of  names 
(TYPEDEFnames and function names (a.k.a.: declared identifiers)) share 
a  name  space,  and  at  every  point  in a source program the (hack) 
contextual distinction can be made by the tokenizer.  Hacks  are  neat 
things in that the less you use them, the more likely they are to work 
when you REALLY need them (i.e., you don't have to fight with existing 
hacks).   Having  worked on designing and implementing a C compiler, I 
was pleasantly amazed at how the constructs all fell together.

The major violations of this approach (i.e., keep declaration separate 
from expressions) that come to mind with C++ are: 

        function-like-casts,
        freestore expressions without parens around the type,
        conversion function names using arbitrary type specifiers,
        parenthesized initializers that drive constructors.

The last problem, parenthesized initializers, provides  two  areas  of 
conflicts.  The first is really an interference issue with old style C 
function  definitions,  which  only bothers folks at file scope (GNU's 
G++ compiler considered this to be too great  an  obstacle,  and  they 
don't currently support old style C definitions!).  The second part of 
this   conflict   involves   a  more  subtle  separation  between  the 
declarator, and the initializer. (K&R  eventually  provided  an  equal 
sign  as  an unequivocal separator, but parens used in C++ are already 
TOO overloaded to separate anything).  The significance of  this  lack 
of  a  clear  separator  is  that  it  is difficult to decide that the 
"declarator" is complete, and that the declared name should  be  added 
to  the scope.  The last problem does interact in a nasty way with the 
function-like cast vs declaration conflicts  (the  problem  slows  the 
feedback  loop  to  the  symbol  table, which is critical to continued 
lexing).  The parened initializers also provide another context  where 
it  is  difficult  to distinguish between expressions (a true argument 
list for the constructor) and a declaration continuation (a  parameter 
type list).  

The  second  problem  listed falls out of the "new-expression" with an 
unparenthesized type.  This form of freestore (such as "new int *  *") 
allows  types  to  be placed adjacent to expressions, and the trailing 
'*' ambiguity rears its head.  I can easily prove  that  this  is  the 
culprit  in  terms  of  specific  ambiguities,  in that removing these 
(unnecessary?) forms significantly disambiguates the grammar.  (It  is 
rather  nice  to  use  YACC  as  a  tool  to  prove  that a grammar is 
unambiguous!).  It is interesting to note that if only the  derivation 
of  a  freestore  expression  were  limited to (using the non-terminal 
names of the form that the C++ Reference manual uses):

        new placement-opt ( type-name )  parened-arg-list-opt

then all the LR(1)  reduce  conflicts  based  on  this  problem  would 
vanish.  Indeed, the culprit can clearly be shown to be:

        new placement-opt  restricted-type-name parened-arg-list-opt

The  characters  which  excite  these reduction conflicts are '*', and 
'&'.

The    third    problem    that    I    indicated     involves     the 
conversion-function-name.   Here  again, if the syntax were restricted 
to ONLY:

        operator simple-type-name

then the LR(1) conflicts would vanish.  It is interesting to note that 
the  keyword  "operator"  serves  as  the  left  separator,  and   the 
restriction   to  "simple-type-name"  results  in  an  implicit  right 
separator  (simple-type-names  are  exactly  one  token  long).    The 
conflicts  appear  when  multiple tokens are allowed for a declaration 
specifier, and an optional pointer-modifier list may  be  added  as  a 
postfix.   The  conflicts  that  result  from  this lack of separation 
include all those provided by the freestore example.

Here again (as with the unambiguous version of freestore)  the  syntax 
could be extended to:

operator_function_name :
        OPERATOR any_operator
        | OPERATOR basic_type_name
        | OPERATOR TYPEDEFname
        | OPERATOR type_qualifier
        | OPERATOR '(' type_name ')'
        ;

instead of:

operator_function_name :
        OPERATOR any_operator
        | OPERATOR type_qualifier_list            operator_function_ptr_opt
        | OPERATOR non_elaborating_type_specifier operator_function_ptr_opt
        ;

and  the  ambiguities  would vanish (and the expressivity would not be 
diminished).


FUNCTION LIKE CAST vs DECLARATION AMBIGUITIES

The real big culprit (i.e., my anti-favorite) in this whole  ambiguity 
set   (re:   keeping   types   and   expressions   separate)   is  the 
function-like-cast.  The reason why it is so  significant  (to  an  LR 
parser)   is  that  the  binding  of  a  type-name,  when  used  in  a 
function-like-cast,  is  directly  to  the   following   parenthesized 
argument list.  In contrast, the binding of a type-name when used in a 
declaration  is to all the "stuff" that follows, up until a declarator 
termination mark like a ',', ';' or '='. Life really gets tough for LR 
folks when the parse stack MUST be reduced, but the parser just  can't 
tell  how  yet.  With this problem, the hacks began to appear (re: the 
"smart lexer").  Note that these new style casts are much more than  a 
notational  convenience  in  C++.   The necessity of the function like 
cast lies in the fact that such a cast  can  take  several  arguments, 
whereas the old style cast is ALWAYS a unary operator.

I was (past tense) actually working towards resolving this problem via 
some  standard  methods that I have developed (re: inline expansion of 
rules to provide deferred reduction).  I was (past tense)  also  using 
one  more  sneaky  piece  of  work  to  defer the reductions, as I was 
carefully making use of right recursion (instead of the standard  left 
recursion)  in  order  to  give  the  parser a chance to build up more 
context.  I can demonstrate the usefulness of right recursive grammars 
in disambiguating difficult toy grammars.  Unfortunately,  I  realized 
at  some  point  that I NEEDED to perform certain actions (such as add 
identifiers to the symbol table) in order  to  complete  the  parse!?!  
This  was  my catch 22.  I could POSSIBLY parse using an LALR grammar, 
if  I  could  only  defer  actions  until  I  had  enough  context  to 
disambiguate.   Unfortunately,  I  HAD to perform certain actions (re: 
modify the symbol table, which changed the action  of  the  tokenizer) 
BEFORE  I  could  continue to examine tokens!  In some terrible sense, 
the typedef hack had come back to haunt me.

I backed off a bit in my grammar after reaching this wall, and now  my 
grammar  only  waits  until  it reaches the identifier in the would be 
declarator.  I really  didn't  want  to  parse  the  stuff  after  the 
identifier  name  (sour  grapes?),  because  I  knew  I would not (for 
example) be able to identify  a  "constant  expression"  in  an  array 
subscript  (most of the time, if it isn't constant, then it can't be a 
declaration).  I don't believe that a compiler  should  compete  in  a 
battle  of  wits  with  the  programmer,  and  the  parser was already 
beginning to outwit me (i.e., I was having a hard time  parsing  stuff 
mentally that is provided as examples in the 2.0 Reference Manual.  In 
fact, many reviewers as well as the author had difficulty, and that is 
why errors appeared there originally, as well as in the ARM).


FUNCTION LIKE CAST vs DECLARATION : THE TOUGH EXAMPLE:        
        
The  following  is about the nastiest example that I have been able to 
construct for this ambiguity group.  I am presenting it here  just  in 
case someone is left with a thought that there is "an easy way out".

The   fact  that  identifiers  can  remain  ambiguous  SO  LONG  after 
encountering them can cause no end of  trouble  to  the  parser.   The 
following  example  does  not  succumb to static (re: no change to the 
symbol table)  anticipatory  lexing  of  a  statement.   As  such,  it 
demonstrates  the  futility  of  attempting  to use a "smart lexer" to 
support the philosophy: "If it can be interpreted  as  a  declaration, 
then so be it; otherwise it is an expression".  This ambiguous example 
exploits  the  fact that declarators MUST be added to the symbol table 
as  soon  as  they  are  complete  (and  hence  they   mask   external 
declarations).

First I will present the example without comments:

class Type1 {
            Type1 operator()(int);
            } ;
class wasType2 { ...}; 
int (*c2)(Type1 dummy);

main ()
    {
    const int    a1  = 0, new_var  (4), (*c1)(int     (a1));
    Type1       (a2) = 3, wasType2 (4), (*c2)(wasType2(a1));
    }




Now to repeat the example with comments:

class Type1 {....
            Type1 operator()(int);
            } ;
class wasType2 { ....}; /* we will almost redeclare this typename!?! */
int (*c2)(Type1 dummy); /* we will NOT redeclare Type1 */

main ()
    {
    /* The first line is indeed simple.  It is simply placed here
    to hint at how the second line MIGHT analogously be parsed. */
    
    const int    a1  = 0, new_var  (4), (*c1)(int     (a1));

    /*  As  a review, "a1" is declared to be a constant with value 0. 
        "new_var" is declared to be another constant, but with  value 
        4.  Finally,  "c1"  is  declared  to  be a pointer to a const 
        integer, and the initial value of this pointer is  "int(a1)", 
        which  is  the  same  as "int(0)", or simply "0" (a.k.a., the 
        null pointer). It is significant that "a1" entered the symbol 
        table  quickly  so  that  it  could  be  used  later  in  the 
        declaration. */

    /* Static lexing of what follows will suggest that the  following 
    is  also  a  declaration.   This  statement  is  actually 3 comma 
    separated expressions!! The explanation that follows  shows  that 
    assuming   the   2nd  statement  is  a  declaration  leads  to  a 
    contradiction. */
    
    Type1      (a2) = 3, wasType2 (4), (*c2)(wasType2(a1));
    
    /* Assume this second statement is a declaration.  Note  that  by 
    the  time  "c2" is parsed, "wasType2" has been redeclared to be a 
    variable of type "Type1".  Hence  "wasType2(a1)"  is  actually  a 
    function  call  to  "wasType2.operator()(a1)",  and  it  is not a 
    function    prototype    arg    list.     It     follows     that 
    "(*c2)(wasType2(a1))"  is  an  expression,  and NOT a declarator!  
    Since this last entry is not a declarator, the  entire  statement 
    must  be an expression (ugh! it is time to backtrack). After much 
    work on my part, I think it might even be  a  semantically  valid 
    expression.   Once this backtracking is complete, we see that the 
    first expression "Type1 (a2) = 3" is  an  assignment  to  a  cast 
    expression.  The second expression "wasType2 (4)", is a cast of a 
    constant.  The  third  expression  "(*c2)(wasType2(a1))",  is  an 
    indirect  function  call.  The argument of the call is the result 
    of a cast.  Note that "wasType2" is  actually  never  redeclared, 
    but it was close! */
    
    /*  For  those of you who can parse this stuff in your sleep, and 
    noticed the slight error  in  the  above  argument,  I  have  the 
    following "fix".  The error is that the 

        "(*c2)(wasType2(a1))" 

    could actually be a declaration with a parenthesized initializer.  
    I could have changed this token subsequence to:

        "(*(*c2)(wasType2(a1)))(int(a1))"

    and avoid the constructor ambiguity, but it would only complicate 
    the  discussion.   Note that in this form, if "wasType2" is not a 
    type, the the quoted text cannot be a declaration.*/

    /* Two parens are all a user would need to  add  to  the  cryptic 
    example  to  unambiguously  specify  that  this  statement  is an 
    expression.  Specifically: */
    
       (Type1) (a2) = 3, wasType2 (4), (*c2)(wasType2(a1));
    /* or ...*/
       (Type1 (a2) = 3), wasType2 (4), (*c2)(wasType2(a1));

    /* I would vote for a syntax error in such ambiguous stream, with 
    an  early  decision that it was a declaration.  After seeing this 
    example, I doubt that I could quickly assert that I could produce 
    a non-backtracking parser that disambiguates statements according 
    to the C++ 2.0 rule.  I am sure  I  can  forget  about  a  simple 
    lex-YACC combination doing it. */

    }


Most  simply  put,  if  a  "smart  lexer"  understands  these: a) I am 
impressed, b) Why use a parser when a lexer can parse so well?

The bottom line is that disambiguation of declarations via "If it  can 
be  a  declaration,  then  it is one", seems to require a backtracking 
parser. (Or some very fancy parsing approach).  I am not even sure  if 
the above examples are as bad as it can get!


CONCLUSION

I believe that the C++ grammar that I have made available represents a 
viable machine readable standard for the syntax description of the C++ 
language.   In  cases  where  the  ambiguities  are  still  exposed by 
conflicts (as noted by YACC), to further  defer  resolution  would  be 
detrimental  to  a  user.   I  see no benefit in describing a computer 
language that must support human writers, but cannot be understood  by 
humans.    Any   code   that  exploits  such  deferral  is  inherently 
non-portable, and deserves to be diagnosed as  an  error  (my  grammar 
asserts a "syntax error").  Rather than dragging the C++ language into 
support  for  a ad-hoc parser implementations such as what cfront (and 
the "smart lexer") have tried unsuccessfully  to  implement,  I  would 
heavily  suggest  the  use  of  my  grammar.  I do not believe that my 
grammar would "break" much existing code, but in cases where it would, 
the code would not be portable anyway (other than  to  a  port  of  an 
IDENTICAL parser).

I  hope  to see a great deal of use of my grammars, and I believe that 
standardizing on the represented syntax will unify  the  C++  language 
greatly.

        Jim Roskind
        Independent Consultant
        516 Latania Palm Drive
        Indialantic FL 32903
        (407)729-4348
        jar@hq.ileaf.com or ...uunet!leafusa!jar



APPENDIX A:
  PROPOSED GRAMMAR MODIFICATIONS (fixing '*', and '&' conflicts)

Based  on  the  other  items  described  above,  I  have the following 
suggestions for cleaning up the grammar definition.  Unfortunately, it 
provides subtle variations from the "C++ 2.0" standard.


Current Grammar:

operator_function_name :
        OPERATOR any_operator
        | OPERATOR type_qualifier_list            operator_function_ptr_opt
        | OPERATOR non_elaborating_type_specifier operator_function_ptr_opt
        ;


operator_new_type:
        type_qualifier_list              operator_new_declarator_opt
                        operator_new_initializer_opt

        | non_elaborating_type_specifier operator_new_declarator_opt
                        operator_new_initializer_opt
        ;

Proposed new grammar (which requires parens around complex types):

operator_function_name :
        OPERATOR any_operator
        | OPERATOR basic_type_name
        | OPERATOR TYPEDEFname
        | OPERATOR type_qualifier
        | OPERATOR '(' type_name ')'
        ;

operator_new_type:
        basic_type_name    operator_new_initializer_opt 
        | TYPEDEFname      operator_new_initializer_opt 
        | type_qualifier   operator_new_initializer_opt 
        | '(' type_name ') operator_new_initializer_opt 
        ;

The impact of the above changes is that all complex type names  (i.e.: 
names  that are not simply a typedef/class name, or a basic type names 
like char) must be enclosed in  parenthesis  in  both  `new  ...'  and 
`operator ...' expressions. Both of the above changes would clear up a 
number  of  ambiguities.   In some sense, the current "disambiguation" 
(of trailing '*', and '&') is really  a  statement  that  whatever  an 
LR(1)  parser cannot disambiguate is a syntax error.  In contrast, the 
above rules define an unambiguous grammar.



APPENDIX B:
	CANONICAL DESCRIPTION OF CONFLICTS and STATES


The following  is  directly  extracted  from  the  canonical  list  of 
conflicts  provided  in  the  y.output  file.   For  a  more  complete 
discussion of the significance of the canonical sentence provided with 
each state, see the autodoc5.txt file.  I have also  added  annotation 
to connect these sentence to the summary given earlier:

state 64: STRUCT IDENTIFIER  .  ':'    (1 reduction, or a shift)
	1 SR caused by member declaration of sub-structure, with trailing :

state 131: OPERATOR INT  .  '*'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 131: OPERATOR INT  .  '&'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 138: OPERATOR CONST  .  '*'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 138: OPERATOR CONST  .  '&'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 281: OPERATOR INT '*' CONST  .  '*'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 281: OPERATOR INT '*' CONST  .  '&'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 282: OPERATOR INT '*'  .  '*'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 282: OPERATOR INT '*'  .  '&'    (1 reduction, or a shift)
	8 SR caused by operator function name with trailing * or &

state 395: CLCL TYPEDEFname '(' TYPEDEFname  .  ')'    (1 reduction, or a shift)
	5 SR caused by redundant parened TYPEDEFname redeclaration vs old style cast
	Make declaration rather than expression.

	problem: Constructor with anonymous arg name at file scope
		looks like redeclaration of typename.

		A::B(C){}  should be the same as  A::B(C x){}  
		(but it isn't for an LR(1) grammar)

state 536: IDENTIFIER '(' '~' TYPEDEFname  .  '('    (1 reduction, or a shift)
	1 SR caused by explicit call to destructor, without explicit scope

state 571: IDENTIFIER '(' NEW INT  .  '*'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 571: IDENTIFIER '(' NEW INT  .  '&'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 572: IDENTIFIER '(' NEW CONST  .  '*'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 572: IDENTIFIER '(' NEW CONST  .  '&'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 621: CLCL TYPEDEFname '(' TYPEDEFname '[' ']'  .  ')'    (1 reduction, or a shift)
	5 SR caused by redundant parened TYPEDEFname redeclaration vs old style cast
	Make declaration rather than expression.
	Don't form an old style cast.

state 738: IDENTIFIER '(' TYPEDEFname '(' ')'  .  ')'    (2 reductions)
	3 RR caused by function-like cast vs typedef redeclaration ambiguity
	LALR-only can be ignored.
	Make declaration rather than expression.

state 738: IDENTIFIER '(' TYPEDEFname '(' ')'  .  ','    (2 reductions)
	3 RR caused by function-like cast vs typedef redeclaration ambiguity
	Problem with LALR-only conflict.
	Make declaration rather than expression.

state 738: IDENTIFIER '(' TYPEDEFname '(' ')'  .  '='    (2 reductions)
	3 RR caused by function-like cast vs typedef redeclaration ambiguity
	Problem with LALR-only conflict.
	Make declaration rather than expression.

state 739: IDENTIFIER '(' TYPEDEFname '(' IDENTIFIER  .  '('    (2 reductions)
	3 RR caused by function-like cast vs identifier declaration ambiguity
	Make declaration rather than expression.

state 739: IDENTIFIER '(' TYPEDEFname '(' IDENTIFIER  .  ')'    (2 reductions)
	3 RR caused by function-like cast vs identifier declaration ambiguity
	LALR-only can be ignored.
	Make declaration rather than expression.

state 739: IDENTIFIER '(' TYPEDEFname '(' IDENTIFIER  .  '['    (2 reductions)
	3 RR caused by function-like cast vs identifier declaration ambiguity
	Make declaration rather than expression.

state 740: IDENTIFIER '(' TYPEDEFname '(' '~' TYPEDEFname  .  '('    (2 reductions)
	3 RR caused by destructor declaration vs destructor call 
	Make declaration of destructor rather than expression.

state 740: IDENTIFIER '(' TYPEDEFname '(' '~' TYPEDEFname  .  ')'    (2 reductions)
	3 RR caused by destructor declaration vs destructor call 
	LALR-only can be ignored.
	Make declaration of destructor rather than expression.

state 740: IDENTIFIER '(' TYPEDEFname '(' '~' TYPEDEFname  .  '['    (2 reductions)
	3 RR caused by destructor declaration vs destructor call 
	Make declaration of destructor rather than expression.

state 758: IDENTIFIER '(' CLCL TYPEDEFname '(' ')'  .  ')'    (2 reductions)
	3 RR caused by parened initializer vs prototype/typename
	Make declaration rather than expression.

state 758: IDENTIFIER '(' CLCL TYPEDEFname '(' ')'  .  ','    (2 reductions)
	3 RR caused by parened initializer vs prototype/typename
	Problem with LALR-only conflict.
	Make declaration rather than expression.

state 758: IDENTIFIER '(' CLCL TYPEDEFname '(' ')'  .  '='    (2 reductions)
	3 RR caused by parened initializer vs prototype/typename
	Problem with LALR-only conflict.
	Make declaration rather than expression.

state 778: IDENTIFIER '(' NEW INT '*' CONST  .  '*'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 778: IDENTIFIER '(' NEW INT '*' CONST  .  '&'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 779: IDENTIFIER '(' NEW INT '*'  .  '*'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 779: IDENTIFIER '(' NEW INT '*'  .  '&'    (1 reduction, or a shift)
	8 SR caused by freestore              with trailing * or &

state 1038: IDENTIFIER '{' TYPEDEFname '(' '(' TYPEDEFname  .  ')'    (1 reduction, or a shift)
	5 SR caused by redundant parened TYPEDEFname redeclaration vs old style cast
	Make declaration rather than expression.

state 1100: IDENTIFIER '{' IF '(' IDENTIFIER ')' ';'  .  ELSE    (1 reduction, or a shift)
	1 SR caused by dangling else and my laziness

state 1102: IDENTIFIER '{' TYPEDEFname '(' '(' TYPEDEFname '[' ']'  .  ')'    (1 reduction, or a shift)
	5 SR caused by redundant parened TYPEDEFname redeclaration vs old style cast
	Make declaration rather than expression.

state 1103: IDENTIFIER '{' TYPEDEFname '(' '*' '(' TYPEDEFname  .  ')'    (1 reduction, or a shift)
	5 SR caused by redundant parened TYPEDEFname redeclaration vs old style cast
	Make declaration rather than expression.

state 1105: STRUCT IDENTIFIER '{' TYPEDEFname '(' TYPEDEFname ')'  .  ';'    (2 reductions)
	6 RR caused by constructor declaration vs member declaration

state 1105: STRUCT IDENTIFIER '{' TYPEDEFname '(' TYPEDEFname ')'  .  '{'    (2 reductions)
	6 RR caused by constructor declaration vs member declaration

state 1152: STRUCT IDENTIFIER '{' TYPEDEFname '(' TYPEDEFname '[' ']' ')'  .  ';'    (2 reductions)
	6 RR caused by constructor declaration vs member declaration

state 1152: STRUCT IDENTIFIER '{' TYPEDEFname '(' TYPEDEFname '[' ']' ')'  .  '{'    (2 reductions)
	6 RR caused by constructor declaration vs member declaration

state 1175: STRUCT IDENTIFIER '{' EXTERN INT '(' TYPEDEFname '[' ']' ')'  .  ';'    (2 reductions)
	6 RR caused by constructor declaration vs member declaration

state 1175: STRUCT IDENTIFIER '{' EXTERN INT '(' TYPEDEFname '[' ']' ')'  .  '{'    (2 reductions)
	6 RR caused by constructor declaration vs member declaration

