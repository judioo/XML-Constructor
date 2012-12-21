[![Build Status](https://travis-ci.org/judioo/XML-Constructor.png)](https://travis-ci.org/judioo/XML-Constructor)

### NAME
**XML::Constructor** - Generate XML from a markup syntax allowing for the
abstraction of markup from code

### SYNOPSIS
A simple example of creating an XML document

         use XML::Constructor;

         my $node  = XML::Constructor->generate(
           parent_node => 'Team',
           data        => [
             {name   => 'Liverpool FC'},
             {league => 'English Premiership'}
           ]
         );

         $node->toString;

The 'toString' method would produce the following XML

         <team>
           <name>Liverpool FC</name>
           <league>English Premiership</league>
         </team>

A more advanced example would be:

         use XML::LibXML;
         use XML::Constructor;

         sub postcode { return { Postcode => 'W11 6TG'} }

         my $surname  = XML::LibXML::Element->new('Surname');
         $surname->appendText('Smith');

         my $element = XML::Constructor->generate(
          parent_node  => XML::LibXML::Element->new('Details'),
          data    => [
            { Forename => 'Joe' },
            $surname,
            [ 'Phone',  mobile  => '0440' ],
            [ 'Phone',  home    => '0441' ],
            [ 'Address',
              [ 'Location',
                type      => 'Home',
                { 'House'   => undef },
                { 'Street'  => '23 Road Street' },
                { 'City'    => 'London' },
                postcode(),
              ],
              [ 'Location',
                type      => 'Work',
                { 'House'   => 'GG&H House' },
                { 'Street'  => '23 Road Street' },
                { 'City'    => 'London' },
                postcode(),
              ],
              { Known_Locations => postcode() }
            ]
          ]
         );

         print $element->toString;

Produces
>          <Details>
>            <Forename>Joe</Forename>
>            <Surname>Smith</Surname>
>            <Phone mobile="0440"/>
>            <Phone home="0441"/>
>            <Address>
>              <Location type="Home">
>                <House/>
>                <Street>23 Road Street</Street>
>                <City>London</City>
>                <Postcode>W11 6TG</Postcode>
>              </Location>
>              <Location type="Work">
>                <House>GG&amp;H House</House>
>                <Street>23 Road Street</Street>
>                <City>London</City>
>                <Postcode>W11 6TG</Postcode>
>              </Location>
>              <Known_Locations>
>                <Postcode>W11 6TG</Postcode>
>              </Known_Locations>
>            </Address>
>          </Details>

### RECOMMEND USER
This package is a wrapper class for [XML::LibXML](https://metacpan.org/module/XML::LibXML) which it uses to
generate the XML.  It provides an abstraction between presentation and
business logic so development of the two can be separated.

This package attempts to satisfy only the most commonly used features
of XML. If you require full DOM specification support (without the
markup separation) there are better packages to use like XML::Generator
of even [XML::LibXML](https://metacpan.org/module/XML::LibXML) directly itself.

That said this package builds and manipulates XML::LibXML instances
which you can always decorate after if you so wished.

### CLASS METHODS
#### generate
         XML::Constructor->generate( parent_node => .. , data => [..] )

* _parameters_: parent_node, data
* _Required_:   none
* _Returns_:    An instance of XML::LibXML::Element [default] |XML::LibXML::Document [if parent_node is an instances of]

'parent_node' can be one of the following

          parent_node ( undef )
                   if not defined a XML::LibXML::Element instance is created with an element name of ""

          parent_node ( XML::LibXML::(Element|Document) )
                   parent_node => XML::LibXML::Element->new('Disco')

                   accepts XML::LibXML::Element or XML::LibXML::Document instances or any object that inherits from either class

           parent_node ( string )
                   parent_node => 'Disco'

                   the string represents the element's name. A XML::LibXML::Element instance is created

           parent_node ( Array ref )
                 parent_node => [ Disco => 'date_start', '1974' ]

                 Will create a new XML::LibXML::Element node as the parent node. The same markup logic used in L<data> is used to build
                 the parent node. This is useful where you have a situation where the parent node also has attributes.

                  The example above will produce a parent node

                   <Disco date_start="1974"/>
                   or

                   <Disco date_start="1974">..</Disco>

                   Depending on whether child nodes are attached. Naturally care must be taken as you can easily be tempted to define
                   complex parent nodes but you should try not to do this! Use L<data> instead.

'data' can be one of the following

           data ( undef )
                 rather pointless but accepted. No markup results in just the parent_node being returned.

           data ( Array ref )
                 containing markup syntax

      toString
         XML::Constructor->toString( parent_node => .. , data => [..] )

       parameters: parent_node, data
       Required:   none
       Returns:    XML output

         convenience method. Wraps generate and calls 'toString' on XML::LibXML::Element|Document instance

### MARKUP SYNTAX
XML::Constructor understands 3 basic types of elements

#### hash:
         { foo => 'bar' }

produces

         <foo>bar</foo>

XML::Constructor takes the key of a hash pairing to be the elements
name. If the value of the pairing is a scalar it is append as text to
the element. The value may also be a non-scalar but this must reference
an array, hash, scalar or a XML::LibXML::Element

Examples:

         { foo => XML::LibXML::Element->new('bar') }

produces

         <foo><bar/></foo>

non-scalar references

         { foo => { bar => 'baz' }}

produces

         <foo>
           <bar>baz</bar>
         </foo>

Also

         { square => \"hat" }

produces

         <square>hat</square>

which is the same as if you passed a normal string. However beware as

         { \"square" => \"hat" }

will produce something similar to
         <SCALAR(0x9a951b8)>hat</SCALAR(0x9a951b8)>

As XML::Constructor will not deference the key.

XML::Constructor supports multi value hashes but note

         { foo => 'bar' , baz => 'taz' }

is NOT equal to

         { foo => 'bar' },{ baz => 'taz' }

As the former does not guarantee order

#### array:
         [ 'foo', bar => 1 ]

produces

         <foo bar='1'/>

When an array is encountered a new instances of XML::LibXML::Element is
created and the 1st value of the array becomes the elements name. The
remaining scalar values of the array become attribute / value pairs
within the element.  References to array, hash, or XML::LibXML::Element
instances are added as child nodes of this element.  References to a
scalar appends the value to the text field of the element.

Examples:

         [ 'foo', { bar => baz } ]

produces

         <foo>
           <bar>baz</bar>
         </foo>

While

         [ 'link', 'rel', 'canonical', 'href', 'http://foo.com', \"lovely foo" ]

urrgh let's add some syntax sugar... While

         [ 'link', rel => 'canonical', href => 'http://foo.com', \"lovely foo" ]

produces

         <link rel="canonical" href="http://foo.com">lovely foo</link>

Naturally care must be taken but you can mix and match the forms quite
safely

         [ 'Phone',
           mobile    => '0440',
           XML::LibXML::Element->new('something'),
           {foo      => 'bar' },
           this      => 'just works',
           \"both text and element :("
         ]

produces

         <Phone mobile="0440" this="just works">
           <something/>
           <foo>bar</foo>
           both text and element :(
         </Phone>

#### XML::LibXML::Element instances
No processing is done. They are simply added to the parent node
#### Code refs
Because of the precedence terms and operators have in Perl it is
possible to embed Perl code into the markup. As long as the term /
function returns valid markup XML::Constructor will not croak.

Here's a simple example:

         sub _count { return map{ {'count'.$_ => " $_"} } (0..shift) }

         XML::Constructor->toString(
           parent_node => 'sequence',
           data        => [ _count(3) ]);

produces

         <sequence>
           <count0> 0</count0>
           <count1> 1</count1>
           <count2> 2</count2>
           <count3> 3</count3>
         </sequence>

This is a powerful feature but much care must be taken. See CAVEATS.

#### scalars ( strings )
strings are appended to the current elements as text. There is an
attempt to remove doubly encoded entities before doing so.

### EXAMPLES
**ORDER MATTERS!**

Adding a string to the top most node

             XML::Constructor->toString(
               parent_node => 'comments',
               data        => [
                 \"1st comment",
                 { 'account', username => 'fuzzbuzz' },
                 \"2nd comment",
                 { 'account', username => 'orth' },
               ]
             );

produces

             <comments>
               1st comment
               <account username="fuzzbuzz"/>
               2nd comment
               <account username="orth"/>
             </comments>

Fibonacci numbers
Non optimal presentation of the sequence

             {
               my %cache = (qw(0 0 1 1));

               sub _fib {
                   my $n = shift;
                   return $n if $n < 2;
                   $cache{$n} = _fib($n -1) + _fib($n - 2);
               }

               sub fibMarkup {
                 my $seed = shift;
                 _fib($seed);
                 return  map{ {'seq'.$_ => " $cache{$_}"} }sort{$a <=> $b} keys %cache;
               }
             }

             my $number = 8;

             print XML::Constructor->toString(
               parent_node   => ['fibonacci', 'sequence' => $number, f0 =>' 0', f1 => ' 1'],
               data    => [ fibMarkup($number) ]);

produces

             <fibonacci sequence="8" f0=" 0" f1=" 1">
               <seq0> 0</seq0>
               <seq1> 1</seq1>
               <seq2> 1</seq2>
               <seq3> 2</seq3>
               <seq4> 3</seq4>
               <seq5> 5</seq5>
               <seq6> 8</seq6>
               <seq7> 13</seq7>
               <seq8> 21</seq8>
             </fibonacci>

### KNOWN ISSUES
Well not really a bug. Rather a gotcha. One thing you can't do is this

         my $ping  = XML::LibXML::Element->new('Ping');
         $ping->appendText('pong');

         print XML::Constructor->toString(
           parent_node => 'missing',
           data        => [
             $ping,
             $ping,
             $ping
           ]
         );

As this will produce

         <missing>
           <Ping>pong</Ping>
         </missing>

and not the expected 3 'Ping' elements. This is an artifact for
XML::LibXML and not this package

### CAVEATS
There are a number of issues this module does not attempt to satisfy.

Using code references within the markup is a powerful feature BUT there
is NO ref counting within the module thus it is possible to fall into a
recursive loop.

There is no native support for namespaces. A half way solution is to
literally code the namespace.

         [ 'rdf:RDF', 'xmlns:rdf' => "http://...", 'rdf:Genre' => 'http://..' ]

produces

         <rdf:RDF xmlns:rdf=".." rdf:Genre=".."/>

but it's not ideal.

There is limited encoding support. The module attempts to identify
double encoding characters but that's it.

If any of these features are deal breakers I advise finding another
package.


### INSTALLATION
To install this module, run the following commands:

  perl Makefile.PL
  make
  make test
  make install

### SUPPORT AND DOCUMENTATION

After installing, you can find documentation for this module with the
perldoc command.

    perldoc XML::Constructor

You can also look for information at:

    RT, CPAN's request tracker
        http://rt.cpan.org/NoAuth/Bugs.html?Dist=XML-Constructor

    AnnoCPAN, Annotated CPAN documentation
        http://annocpan.org/dist/XML-Constructor

    CPAN Ratings
        http://cpanratings.perl.org/d/XML-Constructor

    Search CPAN
        http://search.cpan.org/dist/XML-Constructor/



