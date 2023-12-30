---
author: Bill Bejeck
comments: true
date: 2011-05-19 03:37:12+00:00
layout: post
slug: lucene-thrift-and-ruby
title: Lucene Thrift and Ruby
wordpress_id: 625
categories:
- java
- ruby
- thrift
tags:
- java
- lucene
- ruby
- thrift
---

This post is going to demonstrate thrift usage by searching a Lucene index from Ruby.


### Thrift In a Nutshell


Essentially thrift is a serialization and RPC framework that allows you to communicate between programs that are not necessarily written in the same language.  Thrift is used by defining data types and services in a .thrift file.  You then run the .thrift file against the thrift compiler which generates the stub code needed for clients and servers.  Currently thrift will generate code for C++, C#, Erlang, Haskell, Java, Objective C/Cocoa, OCaml, Perl, PHP, Python, Ruby, and Squeak.  For a more detailed description of thrift along with instructions on how to install thrift if needed,  consult the [thrift wiki](http://wiki.apache.org/thrift/)

<!--more-->

### Generating the Lucene Index


Our first step is to generate a small, simple lucene index.  To build our index, 50,000 fake person records were downloaded from the [Fake Name Generator](http://www.fakenamegenerator.com/order.php) in a comma delimited file.  Each person record will contain a  first name, last name, address and email address.  Our indexing code will be very simple and will not be using any of lucene's advanced features.

    
    
    public class IndexBuilder {
    
        public static void main(String[] args) throws Exception {
            String namesFile = "names.csv";
            Document doc = new Document();
            Field[] fields = new Field[]{new Field("firstName", "", Field.Store.YES, Field.Index.ANALYZED_NO_NORMS),
                    new Field("lastName", "", Field.Store.YES, Field.Index.ANALYZED_NO_NORMS),
                    new Field("address", "", Field.Store.YES, Field.Index.ANALYZED_NO_NORMS),
                    new Field("email", "", Field.Store.YES, Field.Index.ANALYZED_NO_NORMS)};
            addFieldsToDocument(doc, fields);
    
            BufferedReader reader = new BufferedReader(new FileReader(namesFile));
    
            IndexWriter indexWriter = new IndexWriter(FSDirectory.open(new File("blog-index")),new IndexWriterConfig(Version.LUCENE_31, new StandardAnalyzer(Version.LUCENE_31)));
    
            String line;
            while ((line = reader.readLine()) != null) {
                String[] personData = getPersonData(line);
                setFieldData(personData, fields);
                indexWriter.addDocument(doc);
            }
            indexWriter.optimize();
            indexWriter.close();
        }
    
        private static String[] getPersonData(String line) {
            return line.split(",");
        }
    
        private static void setFieldData(String[] data, Field[] fields) {
            int index = 0;
            for (Field field : fields) {
                field.setValue(data[index++]);
            }
        }
    
        private static void addFieldsToDocument(Document doc, Field[] fields) {
            for (Field field : fields) {
                doc.add(field);
            }
        }
    }
    




### Creating the .thrift File 


The next step will be to define what objects and services we want in our .thrift file, which will be called lucene_search.thrift.  The lucene_search.thrift file is intentionally very basic. For more details on the structure of .thrift files consult the [thrift wiki tutorial](http://wiki.apache.org/thrift/Tutorial)

    
    
    //all generated java code will have the following for package name
    namespace java bbejeck.thrift.gen
    
    //this is the person object 
    struct Person {
      1: string firstName,
      2: string lastName,
      3: string address,
      4: string email
    }
    
    //exception used to send meaningful error messages back to user
    exception LuceneSearchException {
      1: string message
    }
    
    //service definition used by client and server
    service LuceneSearch { 
        list<Person> search(1: string query) throws (1:LuceneSearchException error) 
    }
    


As you can see from the example above, the .thrift file format is completely language agnostic. Next we need to generate our java and ruby code.   The following were run from the command line:




  * $ thrift --gen java lucene_search.thrift


  * $ thrift --gen rb lucene_search.thrift


The generated code ends up in two directories named gen-java/ and gen-rb/ respectively. The files generated for java are LuceneSearch.java, LuceneSearchException.java and Person.java.  The generated ruby files are lucene_search.rb, lucene_search_types.rb and lucene_search_constants.rb.   In our next step, we are going to use generated java code to write our thrift server.


### Thrift Server - Java


Thrift generates all the stub code you need for a server to expose your service or program.  The only code we will need to write is a class that implements the generated Iface interface (defined in the LuceneSearch class), which contains the search method defined in our .thrift file.  

    
    
    public class LuceneThriftServer {
        private static final int PORT = 9090;
        private static int numberThreads = 5;
    
        public static void main(String[] args) throws Exception {
            TServerSocket serverSocket = new TServerSocket(PORT, 100000);
            LuceneSearch.Processor searchProcessor = new LuceneSearch.Processor(new SearchHandler(args[0]));
            if (args.length > 1) {
                numberThreads = Integer.parseInt(args[1]);
            }
            TThreadPoolServer.Args serverArgs = new TThreadPoolServer.Args(serverSocket);
            serverArgs.maxWorkerThreads(numberThreads);
            TServer thriftServer = new TThreadPoolServer(serverArgs.processor(searchProcessor).protocolFactory(new TBinaryProtocol.Factory()));
            thriftServer.serve();
        }
    




### Iface Implementation 


The SearchHandler class actually does the work of searching the lucene index.  One tradeoff made here is that any exception while searching is caught and re-thrown as a LuceneSearchException.  While it's usually not a great idea to just re-throw an exception, in this case it makes sense to do so.  Since the LuceneSearchException is defined in the lucene_search.thrift file, the generated client code will handle that exception.  So instead of receiving a generic thrift exception when an error occurs, the client should receive a more meaningful error message. 

    
    
    public class SearchHandler implements LuceneSearch.Iface {
        private IndexSearcher searcher;
        private QueryParser queryParser;
        private static final int MAX_RESULTS = 1000;
    
        public SearchHandler(String indexPath) {
            try {
                searcher = new IndexSearcher(FSDirectory.open(new File(indexPath)), true);
                queryParser = new QueryParser(Version.LUCENE_31, null, new StandardAnalyzer(Version.LUCENE_31));
                queryParser.setAllowLeadingWildcard(true);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    
        public List<Person> search(String query) throws LuceneSearchException {
            List<Person> results = new ArrayList<Person>();
            try {
                Query q = queryParser.parse(query);
                TopDocs topDocs = searcher.search(q, MAX_RESULTS);
                for (ScoreDoc sd : topDocs.scoreDocs) {
                    Document document = searcher.doc(sd.doc);
                    results.add(getPersonFromDocument(document));
                }
            } catch (Exception e) {
                throw new LuceneSearchException(e.getMessage());
            }
            return results;
        }
    
        private Person getPersonFromDocument(Document document) {
            Person p = new Person();
            p.firstName = document.get("firstName");
            p.lastName = document.get("lastName");
            p.address = document.get("address");
            p.email = document.get("email");
    
            return p;
        }
    }
    


The next step in our process is to write the client.


### Thrift Client - Ruby


Writing the thrift ruby client is even easier than writing the server code.  If you have not already done so, install the thrift gem by running "gem install thrift" to get  the required thrift library code.  All the code you need for your client is already generated by thrift.  At this point we are only doing what is needed to get the client to communicate with the server.

    
    
    module ThriftConnection
    
      class LuceneClient
    
        def initialize(host='localhost', port=9090)
          socket = Thrift::Socket.new(host, port)
          @transport = Thrift::BufferedTransport.new(socket)
          protocol_factory = ::Thrift::BinaryProtocolFactory.new
          protocol = protocol_factory.get_protocol(@transport)
          @transport.open
          @client = LuceneSearch::Client.new(protocol)
        end
    
        def search(query)
          @client.search(query)
        end
    
        def close
          @transport.close
        end
    
      end
    end
    




### Running With Scissors


This section has the odd title "Running With Scissors", because like actual running with scissors, what we are about to do may not be a great idea.
In all the thrift generated code there is a warning at the top "DO NOT EDIT UNLESS YOU ARE SURE THAT YOU KNOW WHAT YOU ARE DOING", obviously I don't, but I'm not going to let that stop me at this point (sometimes you just have to see if you can get something to work!)  What we've done is implement method_missing in the generated Person class (found in lucene_search_types.rb) so we can specify searches ala ActiveRecord style.  What we are going to do to accomplish this is use a regular expression to pull out what fields to search for and use the arguments passed in as the search values.  The regular expression here is fairly simple and only aims to handle simple searches.

    
    
    #added to translate from symbol to expected search format
      SEARCH_KEYS_MAPPING = {:first_name => 'firstName',
                                                :last_name => 'lastName',
                                                :email => 'email',
                                                :address => 'address'}
    
    
      def self.method_missing(method_name, *args)
        lucene_client = ThriftConnection::LuceneClient.new
        query = ""
            #handles find_by_first_name etc
        if method_name.to_s =~ /^find_by_([a-z]+_?[a-z]*)$/
          query = "#{SEARCH_KEYS_MAPPING[$1.to_sym]}:#{args[0]}"
            #handles find_by_first_name_or_last_name, find_by_first_name_and_email 
        elsif method_name.to_s =~/^find_by_([a-z]+_[a-z]+)_([a-z]+)_([a-z]+_?[a-z]*)$/
          query ="#{SEARCH_KEYS_MAPPING[$1.to_sym]}:#{args[0]} #{$2.upcase} #{SEARCH_KEYS_MAPPING[$3.to_sym]}:#{args[1]}"
        else
           raise ArgumentError.new("search method pattern #{method_name} not recognized")
        end
    
        results = lucene_client.search(query)
        lucene_client.close
        results
      end
    


As we'll see in the next section, this actually worked, but I still view this more as a useful experiment.  First of all this was placed in generated code, so any time you make changes you would have to manually get the method_missing definition back into the Person class.  Secondly, Lucene search syntax is really not all that hard to learn.  


### Testing


All of what we have done so far would not be worth much if we could not verify our work with some testing.  Here is the unit test to verify that we are indeed able to search a Lucene index from Ruby.  To get some names to search on I simply ran 
    
    
    head names.csv

and then used some of the information in various combinations to get counts of what searches should return.  For example to get an idea of what searching for a first name of Elizabeth or last name of Krause would return I ran
    
    cat names.csv | grep -iE 'elizabeth|krause' | wc -l 

which returned a count of 289.  So, first making sure that our thrift server was running in the background, here is the unit test that was run to verify our Ruby client searching against a Lucene index.

    
    
    class SearchTest < Test::Unit::TestCase
    
      def setup
        @lucene_client = ThriftConnection::LuceneClient.new
      end
    
    
      def teardown
        @lucene_client.close
      end
    
      def test_search_client_first_name
        persons = @lucene_client.search("firstName:Tia")
        assert_equal(5, persons.length)
    
        persons.each do |person|
          assert_equal("Tia", person.firstName)
        end
      end
    
      def test_search_person_class_first_name
        persons = Person.find_by_first_name("Tia")
        assert_equal(5, persons.length)
    
        persons.each do |person|
          assert_equal("Tia", person.firstName)
        end
      end
    
      def test_search_client_first_name_email_domain
        persons = @lucene_client.search("+firstName:Elizabeth +email:*pookmail.com")
        assert_equal(59, persons.length)
      end
    
      def test_search_person_class_first_name_email_domain
        persons = Person.find_by_first_name_and_email("elizabeth", "*pookmail.com")
        assert_equal(59, persons.length)
      end
    
      def test_search_client_first_name_and_last_name
        persons = @lucene_client.search("+firstName:Elizabeth +lastName:Krause")
        assert_equal(1, persons.length)
        person = persons[0]
    
        assert_equal("Elizabeth", person.firstName)
        assert_equal("Krause", person.lastName)
      end
    
      def test_search_person_class_first_name_and_last_name
        persons = Person.find_by_first_name_and_last_name("elizabeth", "krause")
        assert_equal(1, persons.length)
        person = persons[0]
    
        assert_equal("Elizabeth", person.firstName)
        assert_equal("Krause", person.lastName)
      end
    
      def test_search_person_class_first_name_or_last_name
        persons = Person.find_by_first_name_or_last_name("elizabeth", "krause")
        assert_equal(289, persons.length)
      end
    
      def test_invalid_search
        assert_raises ArgumentError do
          Person.find_person_by_name("tia")
        end
      end
    
    end
    




### Conclusion


Thrift is a compelling alternative for RPC or message passing where one might otherwise be using either REST, Java RMI or middleware (JMS, AMQP).  There is a great comparison of how thrift performs against other forms of RPC in this  [thrift tutorial from OCI](http://jnb.ociweb.com/jnb/jnbJun2009.html) found near the end of the article.  It is hoped the reader was able to learn something useful.  Thanks for your time


### Resources


Full source for the blog including the generated code can be found [on github](https://github.com/bbejeck/LuceneThriftRuby). If you are interested in running the test you can download [lucene-thrift-example.tar.gz](https://github.com/downloads/bbejeck/LuceneThriftRuby/lucene-thrift-example.tar.gz) extract the tar file and execute the runSearchTest.sh script.  You do not need to have thrift installed to run the test.




  * For more information on thrift the [ thrift wiki](http://wiki.apache.org/thrift/) is a great start


  * More information on Lucene can be found [here](http://lucene.apache.org/java/docs/index.html)







