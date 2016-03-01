---
layout: post
title:  "Deploy Undertow Server"
date:   2016-02-29 22:59:08
categories: web-service
comments: true

---


#### create maven project

	mvn -B archetype:generate -DarchetypeGroupId=org.apache.maven.archetypes -DgroupId=com.axis.app -DartifactId=server

An `App.java` has been created under `path/src/main/java/com/axis/app/`


#### Add Undertow dependency
{% highlight xml %}
	<dependency>
  		<groupId>io.undertow</groupId>
  		<artifactId>undertow-core</artifactId>
  		<version>1.1.0.CR3</version>
  	</dependency>
  	<dependency>
  		<groupId>io.undertow</groupId>
  		<artifactId>undertow-servlet</artifactId>
  		<version>1.1.0.CR3</version>
  	</dependency>
  	<dependency>
  		<groupId>org.slf4j</groupId>
  		<artifactId>slf4j-log4j12</artifactId>
  		<version>1.7.7</version>
  	</dependency>
{% endhighlight %}

#### incude a plugin to execute App.java class
{% highlight xml %}
    <build>
      <plugins>
         <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>exec-maven-plugin</artifactId>
            <version>1.2.1</version>
            <executions>
               <execution>
                  <goals>
                     <goal>java</goal>
                  </goals>
               </execution>
            </executions>
            <configuration>
               <mainClass>com.axis.app.App</mainClass>
            </configuration>
         </plugin>
      </plugins>
    </build>
{% endhighlight %}	
#### Create undertow hello world service；
{% highlight java %}
	import io.undertow.Undertow;
	import io.undertow.server.*;
	import io.undertow.util.Headers;
	
    public class HelloWorldServer {
    	public static void main(final String[] args) {
        Undertow server = Undertow.builder()
                .addHttpListener(8080, "localhost")
                .setHandler(new HttpHandler() {
                    @Override
                    public void handleRequest(final HttpServerExchange exchange) throws Exception {
                        exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, "text/plain");
                        exchange.getResponseSender().send("Hello World");
                    }
                }).build();
        server.start();
   		}
	}
{% endhighlight %}	
#### Build and run Maven project
{% highlight shell %}
	mvn compile
	mvn exec:java
{% endhighlight %}	

