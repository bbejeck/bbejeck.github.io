---
author: Bill Bejeck
comments: true
date: 2010-02-27 07:43:28+00:00
layout: post
slug: learning-antlr-part-i
title: Learning ANTLR part I
wordpress_id: 72
categories:
- ANTLR
tags:
- ANTLR
- dsl
- java
---

This year one of my goals is to try and become proficient in using ANTLR.  I think that learning to translate text or build an external DSL is skill that, although not used everyday, will be very useful to know. For my first attempt I settled on something fairly easy, a SQL like grammar that could be used to search for files and the content within those files.  You should also be able to narrow the search results based on when the file was last modified.   My goal is to take something like the following:

    
    select * from /logs where file="*.log" and pattern="foobar" and modified < 2 days ago
    select * from /logs where file='.*.log' and pattern='foobar' and modified between 20 and 30 minutes ago
    


and translate it to the corresponding find command and pipe the results to xargs and grep:

    
    find /logs -name '*.out' -mtime -2 | xargs grep 'foobar'
    find /logs -name '*.out' -mmin +20 -mmin -30 | xargs grep 'foobar'
    


As an aside, if you are not familiar with xargs, check out [this xargs tutorial](http://www.cyberciti.biz/faq/linux-unix-bsd-xargs-construct-argument-lists-utility/) or the [xargs man pages](http://man7.org/linux/man-pages/man1/xargs.1.html) , it's a great utility that executes a command with the output of a previous command.

<!--more-->
#### Disclaimer


Now before the villagers gather up with torches and pitch forks to run me out of town (I'm channeling [ Young Frankenstein](http://en.wikipedia.org/wiki/Young_Frankenstein) here), I would like to make somewhat of a disclaimer.  I am not suggesting a new language or discouraging learning the *nix command line tools.  The point here is to learn ANTLR.  I found it more interesting to translate something I use everyday on my current project, versus some of the other "Hello World" ANTLR examples I have seen.  So other than a using this grammar as a learning exercise, I don't see it as being useful.


#### Introduction


ANTLR is a deep topic, so obviously one blog post can not go into any great detail.   So what follows is not in-depth coverage of ANTLR, but a detailed description of the grammar developed.  I will explain each section as well as  some of the decisions and trade-offs I made.  For my development environment I'm using:



	
  1. Eclipse 3.5.1

	
  2. Java 6

	
  3. The [ANTLR IDE ](http://antlrv3ide.sourceforge.net/)plugin for Eclipse.  You could also use [ANTLRWorks](http://www.antlr.org/works/index.html), the gui development environment for ANTLR.  ANTLRWorks is an excellent tool, I just felt more comfortable to do this work in Eclipse.

	
  4. ANTLR version 3.2

	
  5. Mac OS X 10.6.2.


So with all of that out of the way,  let's get started looking at the grammar.


### options, @header



    
    
    grammar FQL;
    options {
         language = Java;
    }
    @header {
         package bbejeck.antlr.fql;
    }
    


Here I am specifying a combined grammar named FQL.  (FQL is short for File Query Language and yes, I know the name sucks)
In options I'm specifying that I want the generated code to be  Java.  I could have also specified C,C++ or Python here as well.  ANTLR also has support for generating code in Ruby, but with the version I am using (v 3.2) I could not get it to work.  I did find [ANTLR Ruby](http://rubyforge.org/projects/antlr3/).  I have not tried it out, but from the documentation it looks promising.  The @header option is setting the package for the generated parser code.  This is also where I would have specified any needed imports.


### @members



The @members section is where you place instance variables and methods that will be placed and used in the generated parser.  Most likely the code in the members section will be used in embedded actions in the parser rules.

    
    
     @members {
      private StringBuilder findBuilder = new StringBuilder("find ");
      
      private StringBuilder filter = new StringBuilder();
      
      private void addString(String s){
        if(s!=null){
            findBuilder.append(s);
         }
      }
      
      private String buildTimeArg(String s, String snum, String sign){
           StringBuilder timeBuilder = new StringBuilder();
           int num = Integer.parseInt(snum);
           
           if(s.equals("days")){
               return timeBuilder.append(" -mtime ").append(sign).append(num).toString();
           }
           if(s.equals("hours")){
               return timeBuilder.append(" -mmin ").append(sign).append((num*60)).toString();
           }
           
           return timeBuilder.append(" -mmin ").append(sign).append(num).toString();
      }
      
      protected void mismatch(IntStream input, int ttype, BitSet follow) throws RecognitionException{
            throw new MismatchedTokenException(ttype,input);
      }
      
      public Object recoverFromMismatchedSet(IntStream input, RecognitionException e, BitSet follow) throws RecognitionException{
         throw e;
      }
      
    }
    


The two StringBuilders _findBuilder_ and _filter_ will be used by embedded actions to build up our translated query.   The reason for two StringBuilders will be explained when we cover the parsing rules.  The _addString_ method is to check for optional tokens that could be null.  I could have easily checked for null in the embedded code within each rule,  but I felt it cluttered the grammar too much.  The _buildTimeArg_ method is used as sort of a poor man's symbol table to translate the _modified_ clause to the proper time format for the _mmin_ or _mtime_ arguments.  
The final two methods override how the generated parser responds to recognition errors (the generated parser extends ANTRL's Parser class which in turn extends the BaseRecognizer class).  By default ANTLR will recover from recognition errors and continue on, trying to read more tokens if available.   But in this grammar, if there is a recognition error along the way I want to stop processing right there.  



### @rulecatch


Each parser rule is converted into a method call in the generated parser with a try - catch block surrounding the parsing code.  The catch statement here will be embedded in each one of the try-catch blocks in the parser.  

    
    @rulecatch{
        catch (RecognitionException e){
                throw e;
          }
    }


If you remember from the previous section we want to stop parsing stop when RecognitionExceptions are encountered, so we re-throw the caught exception.


### @lexer::header


Here we are specifying the package for the generated lexer.

    
    
    @lexer::header {
      package bbejeck.antlr.fql;
    }
    



Now let's move on to the parsing rules.



### Parsing Rules



    
    evaluate returns [String query]
          :  query';' {$query = builder.toString() + filter.toString() ;}
          ;
    
    query
      :   select_stmt where_stmt
      ;
    
    select_stmt
          :  'select' '*' 'from' directory
          ;
    


Here _evaluate_ is our top level rule and returns a String, translated and built as the input is parsed.  Anything within the curly braces is code that will be embedded in the generated parser.  Note how we reference query from the grammar by placing a '$' before the word 'query'.  Also note that the string returned is a concatenation from the two StringBuilders we declared in the @members section.  The _query_ rule is comprised of a _select_stmt_ followed by a _where_stmt_.  The _select_stmt_ is "select * from" followed by the directory rule.

    
    directory
           : (p='.'{addString($p.text);} | (p='/'?{addString($p.text);}IDENT{addString($IDENT.text);})+ )
           ;
    


The directory rule accepts either a '.', a relative or an absolute path.  If the first expression is not provided there must be at least one path expression denoted by the '+'.  The variable 'p' is used to give a handle to the '.' or '/' token so it can be extracted . IDENT is a lexer rule which will be explained a little bit later.  All tokens here are passed into the _addString_ method defined in the members section.

    
    where_stmt
           :  ('where'  clause ('and' clause)* ) ?
           ;
    clause
           : file_name
           | pattern
           | modified
           ;
    


The _where_stmt_ rule expects the string 'where' followed by 0 or more clauses.  Also the entire _where_stmt_  is optional.  Here I chose form over substance.  By that I mean the grammar as it stands here will allow multiple clause's that would not make sense, i.e multiple file_name arguments etc.  I could have specified an exact order of clauses that would have also effectively set the limit of clauses entered, but I would rather the grammar be flexible and trust that the user knows what they want to do.

    
    file_name
           : 'file'  '=' STRING_LITERAL
             {addString(" -name ");addString($STRING_LITERAL.text);}
           ;
    
    pattern
           :   'pattern'  '=' STRING_LITERAL
                 { filter.append(" | xargs grep  ").append($STRING_LITERAL.text); }
           ;
    


The _file_name_ rule sets the -name argument again using the _addString_ method.  The lexer rule STRING_LITERAL will accept whatever the user inputs.  The _pattern_ rule builds up the grep command.  Here we see the use of the second StringBuilder _filter_ that was defined in the @members section.  I feel that having a second StringBuilder to capture text for the grep filter is a hack.   The issue is that the _grep_ command needs to be last in our translated query, but I really want the where statement to be in any order.  So by placing the tokens captured by the _pattern_ rule in a separate StringBuilder I can easily guarantee the _grep_ statement will be last.  

    
    modified
           :  modified_less
           |  modified_more
           |  modified_between
           ;
    


The modified rule has three options.  This portion builds the mmin/mtime argument(s) for the _find_ command.

    
       
              modified_less
                  :   'modified'  '<'  INTEGER time_span                             
                     { addString(buildTimeArg($time_span.text,$INTEGER.text,"-")); }                     
                  ;   
    
             modified_more                     
                  :   'modified'  '>' INTEGER time_span
                      { addString(buildTimeArg($time_span.text,$INTEGER.text,"+")); }
                  ;
    
            modified_between
                  :   'modified' 'between' int1=INTEGER 'and' int2=INTEGER time_span
                     { addString(buildTimeArg($time_span.text,$int1.text,"+")); }
                     { addString(buildTimeArg($time_span.text,$int2.text,"-")); }
                  ;
    


The grammar allows you to specify searching by the time a file was last modified.  Here we use the method _buildTimeArg_ to translate the input to the correct argument for either _mmin_ (minutes modified) or _mtime_ (days modified). Also take note of setting the two variables _int1_ and _int2_.  Those are used to disambiguate which INTEGER token to use.

    
    time_span
           :   'days'
           |   'minutes'
           |   'hours'
           ;
    


The time_span rule allows input of days, minutes or hours.  The hours argument is converted into minutes by the _buildTimeArg_ method.

That's it for the parsing rules, now on to the lexer rules.


### Lexer Rules



    
    fragment DIGIT : '0'..'9';
    fragment LETTER : 'a'..'z'|'A'..'Z' ;
    
    STRING_LITERAL : '\''.*'\'';
    INTEGER : DIGIT+ ;
    IDENT : LETTER(LETTER | DIGIT)* ;
    WS : (' ' | '\t' | '\n' | '\r' | '\f')+  {$channel=HIDDEN;};
    


DIGIT and LETTER are not lexer rules, as you can see by the fragment definition.  These are used for making the grammar more readable.  In the WS definition the {$channel=HIDDEN;} is used to ignore whitespace in the input.



### Test Code


I used the following code to test the grammar from the command line:

    
    public class FQLTester {
    
    public static void main(String[] args) throws Exception{
         BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
         String line = null;
         System.out.println("Enter your search:");
         while((line = reader.readLine())!= null){
             if(line.equalsIgnoreCase("quit")){
                System.exit(0);
             }
            CharStream charstream = new ANTLRStringStream(line);
            FQLLexer lexer = new FQLLexer(charstream);
    
            TokenStream tokenStream = new CommonTokenStream(lexer);
            FQLParser parser = new FQLParser(tokenStream);
    
            String parsed = null;
            try{
                parsed = parser.evaluate();
                System.out.println("parsed query is ["+parsed+"]");
                Process process = Runtime.getRuntime().exec(new String[]{"sh","-c",parsed});
                InputStream input = process.getInputStream();
                BufferedReader procReader = new BufferedReader(new InputStreamReader(input));
                String searchResults = null;
                while((searchResults=procReader.readLine())!=null){
                      System.out.println(searchResults);
                }
            }catch(Exception e){
                   e.printStackTrace();
            }
          System.out.println("Enter your search:");
        }
    }
    



Since this blog is just scratching the surface as far as ANTLR's capabilities are concerned, I plan to be writing more about ANTLR in the near future.  Full source code for everything presented is [available here](http://github.com/bbejeck/antlr_code).
More resources for learning ANTLR are:



	
  * [Scott Stanchfield's video tutorial on ANTLR](http://javadude.com/articles/antlr3xtut/index.html)

	
  * [Definative Guide to ANTLR, Pragmatic Books](http://www.pragprog.com/titles/tpantlr/the-definitive-antlr-reference)



That's it for now, thanks for your time.
