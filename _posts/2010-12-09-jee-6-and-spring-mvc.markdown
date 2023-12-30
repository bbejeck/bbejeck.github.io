---
author: admin
comments: false
date: 2010-12-09 04:56:17+00:00
layout: post
slug: jee-6-and-spring-mvc
title: JEE 6 and Spring MVC
wordpress_id: 356
categories:
- java
- Spring
tags:
- java
- Spring
---

With the release of JEE 6 and the Servlet 3.0 specification came support for asynchronous servlets.  While continuations and Comet are not new, the fact that it is now part of the servlet specification, and could be "baked in" to an application, piqued my curiosity.  Although I have not used plain servlets in development for some time, I have been using Spring MVC.  So I wanted to see what would happen if I added asynchronous support to a Spring-MVC DispatcherServlet.  I created a very simple web application using Spring version 3.0.2 and annotation configuration.  I would like to make clear this is purely an experiement and is not being used production.


### Environment Used
<!--more-->

For this blog I'm using:




  * a T61 Thinkpad running Ubuntu 10.10(64Bit) and 4G ram


  * Java JDK 1.6.0_21


  * Glassfish Version 3. (I tried Tomcat 7 and Jetty 8, but had the best luck at this point with Glassfish)




### DispatcherServlet Code


My first step was to extend Spring's DispatcherServlet.

    
    @WebServlet(urlPatterns = {"/async/*"}, asyncSupported = true, name = "async")
    public class AsyncDispatcherServlet extends DispatcherServlet {
        private ExecutorService exececutor;
        private static final int NUM_ASYNC_TASKS = 15;
        private static final long TIME_OUT = 10 * 1000;
        private final Log log = LogFactory.getLog(AsyncDispatcherServlet.class);
    @Override
        public void init(ServletConfig config) throws ServletException {
            super.init(config);
            exececutor = Executors.newFixedThreadPool(NUM_ASYNC_TASKS);
        }
    
        @Override
        public void destroy() {
            exececutor.shutdownNow();
            super.destroy();
        }
    
        @Override
        protected void doDispatch(final HttpServletRequest request, final HttpServletResponse response) throws Exception {
            final AsyncContext ac = request.startAsync(request, response);
            ac.setTimeout(TIME_OUT);
            FutureTask task = new FutureTask(new Runnable() {
    
                @Override
                public void run() {
                    try {
                        log.debug("Dispatching request " + request);
                        AsyncDispatcherServlet.super.doDispatch(request,response );
                        log.debug("doDispatch returned from processing request " + request);
                        ac.complete();
                    } catch (Exception ex) {
                        log.error("Error in async request", ex);
                    }
                }
            }, null);
    
            ac.addListener(new AsyncDispatcherServletListener(task));
            exececutor.execute(task);
        }


The only methods overridden were init, destroy and doDispatch. I won't go into detail on init and destroy, what they do is obvious.  All the interesting work is done in doDispatch.  The doDispatch method starts the asynchronus request, then wraps the call to super.doDispatch in a runnable and passes that into an executor service. There are a few key points to consider here:



        
  1. The @WebServlet annotation at the class definition level.  This is part of Servlet 3.0 specification, you can now declare servlets, filters etc via annotations, although you can still use web.xml.  To enable asynchronous support set the 'asyncSupported' attribute to true.

	
  2. On line 22 the setting of a timeout for the asyncContext object. In this case the timeout is 10 seconds

	
  3. On line 38 setting an AsyncContextEventListener.

        
  4. The application server thread returns almost immediately.




### Listener Code


Getting the request to run asynchronously is only half the battle.  The other half is setting up hooks to handle different events during the life-cycle of the asynchronous request.  The Servlet 3.0 spec added the AsyncListener interface.  AsyncListener has 4 methods, onStartAsync, onComplete, onError and onTimeout. For the AsyncDispatcherServlet we have the inner class AsyncDispatcherServletListener  that takes a FutureTask object as a constructor argument.

    
    private class AsyncDispatcherServletListener implements AsyncListener {
    
            private FutureTask futureTask;
    
            public AsyncDispatcherServletListener(FutureTask futureTask) {
                this.futureTask = futureTask;
            }
    
            @Override
            public void onTimeout(AsyncEvent event) throws IOException {
                log.warn("Async request did not complete timeout occured");
                handleTimeoutOrError(event, "Request timed out");
            }
    
            @Override
            public void onComplete(AsyncEvent event) throws IOException {
                log.debug("Completed async request");
            }
    
            @Override
            public void onError(AsyncEvent event) throws IOException {
                log.error("Error in async request", event.getThrowable());
                handleTimeoutOrError(event, "Error processing " + event.getThrowable().getMessage());
            }
    
            @Override
            public void onStartAsync(AsyncEvent event) throws IOException {
                log.debug("Async Event started..");
            }
    
            private void handleTimeoutOrError(AsyncEvent event, String message) {
                PrintWriter writer = null;
                try {
                    future.cancel(true);
                    HttpServletResponse response = (HttpServletResponse) event.getAsyncContext().getResponse();
                    //HttpServletRequest request = (HttpServletRequest) event.getAsyncContext().getRequest();
                    //request.getRequestDispatcher("/app/error.htm").forward(request, response);
                    writer = response.getWriter();
                    writer.print(message);
                    writer.flush();
                } catch (IOException ex) {
                    log.error(ex);
                } finally {
                    event.getAsyncContext().complete();
                    if (writer != null) {
                        writer.close();
                    }
                }
            }
        }


The onStartAsync and onComplete methods merely log a statement, but certainly could be used to open and close resources respectively.  The only methods that do any work are onTimeout and onError, delegating to the handleTimeoutOrError method, passing a message and the AsyncEvent object.  In handleTimeoutOrError we will call cancel on the futureTask object, write the message to the response stream, then mark the asyncContext as completed.  While we are writing the error directly to the response stream, we could have just as easily forwarded to an error page by using the commented out call to request.getRequestDispatcher().forward (obviously you would eliminate lines 38-40).


### Web Application Structure





This web application is very simple and has only two controllers - SimpleViewControler and SearchController.

    
    
    
    @Controller
    public class SimpleViewController {
        
        @RequestMapping({"/","/index.htm"})
        public String showHome(){
            return "index";
        }
    
        @RequestMapping({"/error.htm"})
        public String error(){
            return "error";
        }
    




    
    
    @Controller
    public class SearchController {
    
        @RequestMapping("/search.htm")
        public String doSearch(@RequestParam(value = "latency", defaultValue = "2000") long latency,
                                            @RequestParam(value = "blowup", defaultValue = "false") boolean blowUp,
                                            Model model) throws Exception {
    
            String searchResult = getSearchResult(latency, blowUp);
    
            model.addAttribute("result", searchResult);
            return "searchResult";
        }
    
        @RequestMapping("/search.ajax")
        public void doSearchAjax(@RequestParam(value = "latency", defaultValue = "2000") long latency,
                @RequestParam(value = "blowup", defaultValue = "false") boolean blowUp,
                HttpServletResponse response) throws Exception {
    
            String searchResult = getSearchResult(latency, blowUp);
            
            PrintWriter writer = null;
            try {
                writer = response.getWriter();
                writer.print(searchResult);
                writer.flush();
            } finally {
                if (writer != null) {
                    writer.close();
                }
            }
        }
    
        private String getSearchResult(long latency, boolean blowUp) throws Exception {
    
            if (blowUp) {
                throw new RuntimeException("Bad error happened in controller");
            }
    
            Thread.sleep(latency);
    
            StringBuilder builder = new StringBuilder("Some search/whatever results being returned");
            Date now = new Date();
            builder.append(" @").append(now);
            
            return builder.toString();
        }
    


The latency and blowup parameters in SearchController are used to simulate different response times and errors respectively.  Although there is the doSearchAjax method which writes directly to the response stream, in the tests that are run, we will only be using the doSearch method.  There are the usual context files, which are very light do to the annotation configuration and a web.xml file (needed for the regular DispatcherServlet). 








### Testing


 Now it's time to see if this experiment works at all.   JMeter is a great tool and it is what I used to load test our simple web application.  I have set up three tests.




  1. A "control" test - There are two thread-groups consisting of 50 threads each and will ramp up to run all threads in 3 seconds.  One thread group will make requests to /app/index.htm and the other thread group will make requests to /app/search.htm.  The thread-groups will execute simultaneously  and loop 3 times for a total of 300 requests.  Each thread-group has a "listener" attached to it to measure throughput, and there is a listener attached to the test to measure overall throughput. This test will give us our baseline.  The requests to /app/search.htm will not set any parameters, so each request will have the default value of 2 seconds for latency.



  2. The "aynscronous" test - This test is will measure the effect of using asynchronous servlets in the application.  Setup is identical to the control test above with one exception - the search requests will go to /async/search.htm and hit the AsynchronousDispatcherServlet.



  3. An error condition test - This test will be structured a little differently.  The thread-group for /app/index.htm has a longer ramp up time, but will remain the same otherwise.  The thread group for /async/search.htm will add a JMeter option known as a 'RandomController'.  There will be 3 possible search requests sent, a valid request, a request with the latency parameter set 12 seconds causing a timeout and a request with the blowup parameter set to true, so a RuntimeException will be thrown.





### Test Results
















Control Test




Request
Throughput





Overall


296.174/minute






Index


164.489/minute






Search


148.569/minute













Asyncronous Test



  RequestThroughput






   
Overall
849.177/minute





   
Index
2,909.796/minute





   
Search
429.84/minute













Error Test



  RequestThroughput






   
Overall
115.848/minute





   
Index
2,803.738/minute





   
Search
57.974/minute










As we can see from the test results, sending the search requests through the AsyncronousRequestDispatcher increased application throughput.  The request per minute numbers don't mean that much though, given that the web application was so simple and the test was very contrived. What matters more is that the index requests had roughly the same response time and were seemingly unaffected when asynchronous support was used for the search requests




### Summary




For me there were two main takeaways from this experiment:




  * Even though asyncronous support seemed help with throughput, it is still using a thread pool which consumes server resources, so it should be only be applied to very select parts of an application.


  * By setting timeouts and getting a chance to handle them gracefully via the event listener, asynchronous support acts a "circuit breaker" of sorts.  This could be valuable when your application makes requests to outside resources that may be down or otherwise unresponsive.







### Resources




Source for everything is [available on github](http://github.com/bbejeck/spring_servlet3).  

To run the JMeter tests



  1. [Download JMeter](http://jakarta.apache.org/site/downloads/downloads_jmeter.cgi) and extract the tar/zip file to some directory


  2. Copy all of the *.jmx files in the jmeter directory from the github site for the code into <JMeter install>/bin. The from the bin directory run jmeter or jmeter.bat depending on your platform.  Once JMeter is up and running select File and you should see AsyncWebTestControl.jmx, AsyncWebTestErrors.jmx, AsyncWebTest.jmx in the File menu.   Just click on one of those to open then Ctrl+r to run a test


  3. Download the war file and deploy to glassfish.  I placed the war file in the autodeploy directory in glassfish.  On my laptop it's in /usr/local/servers/glassfishv3/glassfish/domains/domain1/autodeploy.



