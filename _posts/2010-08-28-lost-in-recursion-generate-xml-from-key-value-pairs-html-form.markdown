---
layout: post
title: Lost in Recursion - Generate XML from key value pairs (HTML Form)
---
# Lost in Recursion - Generate XML from key value pairs (HTML Form)
## Meta

* Pre-requisites
> MarkLogic Server 4.2.
> Knowing your way around http://localhost:8001 or a working cq instalation
* Difficulty: Medium
* Keywords: XML, XForms, HTML, Forms, XQuery, Functional, Map, Fold, High Order Functions, Application Buider, Search, git, github, XQuery 1.1

## Introduction
One of the cool things about XForms is that I can abstract the data model from the form and get a consistent view of my XML. For me this is the killer feature about XForms. However, regular HTML forms are way more pervasive and I found myself thinking on how I could implement this feature in standard HTML.

In XForms we have a model (which is XML) and also a form that acts on that model. So our form "knows" the XML structure. In HTML forms there's no notion of data model implicit, or anything like that. What is submitted from an HTML form is a set of key value pairs.

![HTML Form](http://img.skitch.com/20100813-bufb2yatuw94uwisk7g2m8im1x.png)

In this little article we are going to design an application that can insert and search multiple choice questions using HTML. The HTML form will be responsible for the insert. The search will be tackled with Application Builder in part two of this article.

## Part 1: Creating the Form
For the sake of this demonstration let's assume 'option_a' is always the correct option, thus avoiding another control. This is ok as we can randomize this list in the server side once we receive the options.

So while in XForms we would submit something like:
  <question>
    <text>Which of the following twitter users works for MarkLogic?</text>
    <answer>
      <a>peteaven</a>
      <b>jchris</b>
      <c>stuhood</c>
      <d>antirez</d>
    </answer>
  </question>

In regular html you have something like:
  > POST / HTTP/1.1
  > Content-Type: application/x-www-form-urlencoded
     question=Which of the following twitter users works for MarkLogic?
     &option_a=peteaven&option_b=jchris&option_c=stuhood&option_d=antirez

While this can map perfectly to a relational database it doesn't play well with XML. Let me rephrase this: There are multiple ways you could shape it as XML.

One possible solution is to name the fields with an XPath expression and then generate an XML tree out of this path expression.

![HTML Form with XPath expressions as field names](http://img.skitch.com/20100813-k83ufc24sgdkjmm6wuhxq746sc.png)

Once we solve this we have two options on how to generate the XML from XPath: Do some work with a client-side language like javascript and produce the XML that is sent to the server or simply submit the form and create the XML on the server-side with XQuery. I choose the second approach for two reasons:

  1. To push the XQuery High Order Functions support in MarkLogic Server to the limit and learn how far it can go.
  2. Other people might have a similar problem that needs to be solved in the server side. This way they can reuse the code.

High order functions are functions that take functions as parameters. 

>> Geek Corner
>> Ever heard of MapReduce? 
>> This is a generally accepted paradigm in distributed computing first published by google. 
>> However the functions map and reduce are just the names of the high order functions that are used in the paper. 
>> In mathematical terms it should actually be 'reduce . map' but compose functions and point-free notation are well behind scope here (just as out of scope as they are awesome). 
>> I still wonder why no-one thought of 'reduce . map . filter' yet and using the filter stage to introduce basic lookup indexes

Two examples of such functions are fold (a.k.a. reduce or inject) and map (a.k.a. collect or transform). 

Fold is a list destructor. You give it a list l, a starting value z and a function f. Then the fold starts accumulating the value of applying f to each element of l in z. Map is a function that applies a function f to each element of a list.

An example of a fold might be implementing sum, a function that sums the contents of a list:

    # in no particular language, pseudo code
    sum l = fold (+) 0 l

An example of a map is multiply every element in a list by two:

    # in no particular language, pseudo code
    double l = map (2*) l
  
>> Geek Corner
>> A fold is really just a list destructor. But you can generalize it for any  arbitrary algebraic data types. 
>> You call these "generic folds" a catamorphism. Actually a fold is just a catamorphism on lists.

Implementing these functions in MarkLogic XQuery 1.0 with recursion is really easy:

    declare function local:head( $l ) { $l[1] } ;
    declare function local:tail( $l ) { fn:subsequence( $l, 2 ) } ;
    declare function local:fold( $f, $z, $l ) { 
      if( fn:empty( $l ) ) then $z
      else local:fold( $f,
                       xdmp:apply( $f, $z, local:head( $l ) ),
                       local:tail( $l ) ) } ;

    declare function local:map( $f, $l ) {
      for $e in $l return xdmp:apply( $f, $e ) } ;

    declare function local:add($x, $y)         { $x + $y } ;
    declare function local:multiply($x, $y)    { $x * $y } ;
    declare function local:multiply-by-two($x) { $x * 2 } ;

    (: sums a list using fold :)
    declare function local:sum( $l ) {
      let $add      := xdmp:function( xs:QName( 'local:add' ) )
      return local:fold( $add, 0, $l ) } ;

    declare function local:double ( $l ) {
      let $multiply-by-two := 
        xdmp:function( xs:QName( 'local:multiply-by-two' ) )
      return local:map( $multiply-by-two, $l ) } ;

    (: factorial just for fun :)
    declare function local:fact($n) { 
      let $multiply := xdmp:function(xs:QName('local:multiply'))
      return local:fold($multiply, 1, 1 to $n) };

    (: This is the main part of the XQuery file
     : Illustrating the fold and map from the previous listing :)
    <tests>
      <!-- fun facts: http://www.mathematische-basteleien.de/triangularnumber.htm -->
      <sum> { local:sum(1 to 100) } </sum>
      <fact> { local:fact( 10 ) } </fact>
      <double> { local:double( (1 to 5) ) } </double>
    </tests>

>> Geek Corner
>> Good news is XQuery 1.1 will have improved support for High Order Functions and that will be awesome. 
>> No more xdmp:apply and it will be directly integrated in the syntax. 

So how can we use all of this to solve our XPath to XML problem? Simple. We need to destruct the list of xpaths and generate a tree. In other words, we need to fold the list intro a tree.

![Generating a tree](http://img.skitch.com/20100814-m3kg2qiadmhpt1abtq8kwsejfe.png)

If we go one level down an XPath is really a list of steps. Once again we need to destruct that list to create each node. So we need a fold inside a fold.

We now need to iterate the list of field values, navigate to the corresponding node using the XPath expression, and finally replace the value of the node (empty at this point) with the value provided in the HTTP form.

Scared? Wondering if we really need all this functional stuff? Fear not, problem is solved and we will simply use a XQuery library module that already exists to solve the problem! Hooray. 

The library is called generate-tree and is included in the dxc github project. To get it simply install git and:

    git clone git://github.com/dscape/dxc.git

If you don't know what git is (neither you care) simply go to the project page at http://github.com/dscape/dxc and download the source.

If you are curious to see the implementation using the folds and everything you learned so far you can check the the [gen-tree.xqy](http://github.com/dscape/dxc/blob/master/func/gen-tree.xqy) implementation at github. Or as an exercise you can try and do it yourself! To run this code directly from cq I created [another script](http://gist.github.com/364356) that creates a tree while printing out debug messages. This might be useful to understand how the code is running without getting "lost in recursion".

Create a folder called 'questions-form' and place the dxc code there:
    njob@ubuntu:~/Desktop/questions-form$ ls -l
    total 8
    drwxr-xr-x 12 njob njob 4096 2010-08-13 20:51 dxc
    -rw-r--r--  1 njob njob  149 2010-08-13 20:59 index.xqy

Now we need to create the HTML form. For now simply create a file called index.xqy inside the 'questions-form' directory and insert the following code:

    xquery version '1.0-ml';

    "Hello World!"

In this listing we simply print Hello World! To get our website online simply go the the MarkLogic Server Administration Interface at http://localhost:8001 and create a new Application Server with the following parameters:

    name: questions-form
    port: <any port that is available in your system>
    root: <full path of the directory where you have the index.xqy file>

In my case this will be:

    port: 6173
    root: /home/njob/Desktop/questions-form

If you have cq installed you can simplify the process by running the following script (remember to change the root. Also change the port if necessary)

    xquery version '1.0-ml';

    import module namespace admin = "http://marklogic.com/xdmp/admin" 
      at "/MarkLogic/admin.xqy" ;

    let $name       := "questions-form"
    let $root       := "/home/njob/Desktop/questions-form"
    let $port       := 6173
    let $config     := admin:get-configuration()
    let $db         := "Documents"
    let $groupid    := admin:group-get-id( $config, "Default" )
    let $new        := admin:http-server-create( $config, $groupid, $name, 
      $root, xs:unsignedLong( $port ), 0, xdmp:database( $db ) )
    return ( admin:save-configuration( $new ) ,
             <div class="message">
               An HTTP Server called {$name} with root {$root} on 
               port {$port} created successfully </div> )

This is running against the default Documents database. This is ok for a demonstration but in a realistic scenario you would be using your own database.

Now when you visit http://localhost:6173 you will get a warm Hello World! 

Now let's change the code to actually perform the transformation. To do so simply insert this code in index.xqy. Feel free to inspect it and learn from it - I commented it just for that reason.

    xquery version '1.0-ml';
    
    (: First we import the library that generates the tree :)
    import module namespace mvc = "http://ns.dscape.org/2010/dxc/mvc"
      at "dxc/mvc/mvc.xqy" ;

    (: 
     : This function receives a string as the parameter $o
     : which will be either 'a', 'b', 'c' or 'd' and
     : generates an input field for the form
     :)
    declare function local:generate-option( $o ) {
     (<br/>, <label for="/question/answer/{$o}">{$o}) </label>,
          <input type="text" name="/question/answer/{$o}" 
            id="/question/answer/{$o}" size="50"/>) };

    (: This function simply displays an html form as described in the figures :)
    declare function local:display-form() {
      <form name="question_new" method="POST" action="/" id="question_new">
        <label for="/question/text">Question</label><br/>
        &nbsp;&nbsp;&nbsp; <textarea name="/question/text" id="/question/text" 
          rows="2" cols="50">
        Question goes here </textarea>
      <br/>
      { (: using the generate option function button to generate four fields :)
        for $o in ('a','b','c','d') return local:generate-option( $o ) }
      <br/><br/><input type="submit" name="submit" id="submit" value="Submit"/>
       </form> } ;
    
    (: this function will process the insert and display the result
     : for now it simply shows the tree that was generated from the HTML form
     :)
    declare function local:display-insert() {
      xdmp:quote( mvc:tree-from-request-fields() ) } ;
    
    (: Now we set the content type to text html so the browser renders
     : the page as HTML as opposed to XML :)
    xdmp:set-response-content-type("text/html"),
    <html>
      <head>
        <title>New Question</title>
      </head>
      <body> {
      (: if it's a post then the user submited the form :)   
      if( xdmp:get-request-method() = "POST" )
      then local:display-insert()
      else
        (: the user wants to create a new question :)
        local:display-form() }
      </body>
    </html>

We are using the 'mvc:tree-from-request-fields()' function to create the tree from the request fields. However this function isn't described in [gen-tree.xqy](http://github.com/dscape/dxc/blob/master/func/gen-tree.xqy). This is declare in another library called mvc: 

    declare function mvc:tree-from-request-fields() {
      let $keys   := xdmp:get-request-field-names() [fn:starts-with(., "/")]
      let $values := for $k in $keys return xdmp:get-request-field($k)
      return gen:process-fields( $keys, $values ) } ;

Now you can visit http://localhost:6173 again and you'll see our form. Fill it accordingly to the following picture and click "Submit"

![How the forms looks like](http://img.skitch.com/20100814-j5j3yn3ny21ne7fuaqed854g8i.png)

This is how the document you inserted looks like:

    <?xml version="1.0" encoding="UTF-8"?>
    <question>
      <text>Which of the following twitter users works for MarkLogic?</text>
      <answer>
        <a>peteaven</a>
        <b>jchris</b>
        <c>stuhood</c>
        <d>antirez</d>
      </answer>
    </question>

Now let's augment our form with some more interesting fields like author and difficulty. This will help make our search application interesting.
Simply update the display-form function:
    
    (: This function simply displays an html form as described in the figures :)
    declare function local:display-form() {
      <form name="question_new" method="POST" action="/" id="question_new">
        <input type="hidden" name="/question/created-at" 
          id="/question/created-at" value="{fn:current-dateTime()}"/>
        <input type="hidden" name="/question/author" 
          id="/question/author" value="{xdmp:get-current-user()}"/>
        <br/> <label for="/question/difficulty">Difficulty: </label>
          <input type="text" name="/question/difficulty" 
            id="/question/difficulty" size="50"/>
        <br/> <label for="/question/topic">Topic:&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
        </label>
          <input type="text" name="/question/topic" 
            id="/question/topic" size="50"/>
        <br/><br/> <label for="/question/text">Question</label><br/>
        &nbsp;&nbsp;&nbsp; <textarea name="/question/text" id="/question/text" 
          rows="2" cols="50">
        Question goes here </textarea>
      <br/>
      { (: using the generate option function button to generate four fields :)
        for $o in ('a','b','c','d') return local:generate-option( $o ) }
      <br/><br/><input type="submit" name="submit" id="submit" value="Submit"/>
       </form> } ;

The end result should be this form:

![Our form is complete](http://img.skitch.com/20100814-mj3betss6tiq143tq9gpsdhi72.png)

Now we are missing the part where we actually insert the document in the database. For that we need to update the function that local:display-insert() function:

    (: this function will process the insert and display the result
     : it then redirects to / giving you the main page
     :)
    declare function local:display-insert() {
      try {
        let $question   := mvc:tree-from-request-fields() (: get tree :)
          let $author     := if ($question//author[1]) 
                             then fn:concat($question//author[1], "/") else ()
          (: now we insert the document :)
          let $_          := xdmp:document-insert(
            (: this fn:concat is generating a uri with directories
             : e.g. /questions/njob/2362427670145529782.xml 
             :)
            fn:concat("/questions/", $author, xdmp:random(), ".xml") , $question )
          return  xdmp:redirect-response("/?flash=Insert+OK")
      } catch ($e) {
        xdmp:redirect-response(fn:concat("/?flash=", 
          fn:encode-for-uri($e//message/text()))) } } ;

So far we talked about the problem, differences with XForms, proceeded to talk on high order functions and how to implement it in XQuery and finally we got a working solution for our little problem. Coming up next we are going to build an application to search these questions we can now insert with Application Builder. Then we are going to take advantage of the new functionalities available in MarkLogic 4.2. to extend application builder with this form. 