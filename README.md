Mashup Query Language
=====================

MQL is a data query language that seeks to make it easy to do:
1) Filtering of complex data sets
2) Transformation of objects
3) Join data across disparate data sources

Using the Query Language
------------------------
Executing queries is simple:

	// populate some data
	List<User> users = new ArrayList<User>();
	users.add(new User("Dan", "Diephouse", "MuleSoft", "Engineering"));
	users.add(new User("Joe", "Sales", "MuleSoft", "Sales"));
	
	// create a context for the query
	Map<String,Object> context = new HashMap<String,Object>();
	context.put("users", users);
	
	// execute the query
	Collection<User> result = 
	    Query.execute("from users where division = 'Engineering'", context);
	  
	assertEquals(1, result.size()); // the result will just contain the first user

The query syntax is best illustrated by example queries below.

Filtering Collections:

	from people where division = 'Sales'
	from people as u where u.division = 'Sales' // explicit syntax
	from people where division = 'Sales' and (firstName = 'Dan' or firstName = 'Joe')
	from people where division = 'Sales' 

Querying objects in your query context:
   
    // execute the getPeople() method on the salesforce object
    from salesforce.people as u where u.lastName = 'Diephouse'
    // the more explicit syntax is also valid
    from salesforce.getPeople() as u where u.lastName = 'Diephouse'
    
Ordering collections:

	from people as u order by name

Transforming a collection into new objects:

    select new {
	  href = 'http://localhost/sales/people/' + id,
	  name = firstName + ' ' + lastName,
	  division = division
	}
	
	// This example shows the more explicit syntax
	from users as u select new {
	  href = 'http://localhost/sales/people/' + u.id,
	  name = u.firstName + ' ' + u.lastName,
	  division = u.division
	}

These queries will create a new set of objects with href and name properties. 
The href property will be a combination of the string url and the object id. 
The name property will be a synthesis of the first and last name from the 
user object.

Joining a data source:

   from people as u 
     join twitter.getUserInfo(u.twitterId) as twitterInfo
     select new {
       name = u.firstName + ' ' + u.lastName,
       tweets = twitterInfo.totalTweets
     }

This query assumes that you have an object named "twitter" inside your 
query context which has a method called getUserInfo() which takes a twitter 
user id. It then joins the result of this method call into a new object 
called "twitterInfo" which can be used in the select statement.

If you do a join, they happen asynchronously via an Executor. You can specify 
the number of threads you wish to use to do the join (i.e. the number of rows 
you want to process simultaneously) via the async keyword. E.g. this will use
10 threads to do the join:

    from people as u
       join twitter.getUserInfo(u.twitterId) as twitterInfo async(10) ...       

You can also control when the join happens via the on keyword:

    from people as u
       join twitter.getUserInfo(u.twitterId) as twitterInfo on u.twitterId ...       

This will ensure the join happens only when there is a twitterId property 
on the user.

To handle the a null response fro the join method, you can do the following:

    from people as u
       join twitter.getUserInfo(u.twitterId) as twitterInfo 
       select new {
         name = u.name,
         tweets = twitterInfo.?tweets
       }

This will ensure that your query will continue to work even if twitterInfo 
is null for a particular user.


Using MQL in Mule
=================
There are two ways to use MQL in Mule. First, you can create a MQL query service.
This is handy for turning existing data (e.g. cloud connectors) into services
very quickly. For example, this simple config will create a new JSON data
service for your twitter cloud connector:

	<?xml version="1.0" encoding="UTF-8"?>
	<mule xmlns="http://www.mulesoft.org/schema/mule/core" 
	    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	    xmlns:spring="http://www.springframework.org/schema/beans"
	    xmlns:mql="http://www.mulesoft.org/schema/mule/mql" 
	    xsi:schemaLocation="
	               http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
	               http://www.mulesoft.org/schema/mule/core http://www.mulesoft.org/schema/mule/core/3.1/mule.xsd
	               http://www.mulesoft.org/schema/mule/mql http://www.mulesoft.org/schema/mule/mql/3.1/mule-mql.xsd
	               ">
	    <twitter:config id="twitter" ... />
	    
	    <mql:query-service
	         name="recentMuleTweets"
	         address="http://localhost:8080/recentMuleTweets"
	         query="from twitter.userTimeline where message like 'mule'" />
	
	</mule>

There is an optional type attribute which can be set to "POJO" or "JSON" depending
on what data you want to return. For example, to work with POJOs you might configure
the following:

	    <mql:query-service
	         name="recentMuleTweets"
	         type="POJO"
	         address="vm://recentMuleTweets"
	         query="from twitter.userTimeline where message like 'mule'" />
	
You can also use the mql:transform element inside of Mule to do MQL transformations.
Here is a very simple flow:

    <flow name="mql">
        <inbound-endpoint address="vm://select"
            exchange-pattern="request-response" />
        <mql:transform query="from payload where division = 'Sales'
                                select new { name = firstName + ' ' + lastName }" />
    </flow>

If you want, you can refer to properties inside the Mule Message inside the 
where or select statements. In this example, the property 'someData' is 
added to the message and then added to the 'data' field of the object that
is created via the select statement.

    <flow name="selectDataFromMessageProperty">
        <inbound-endpoint address="vm://join"
            exchange-pattern="request-response" 
        <message-properties-transformer scope="invocation">
            <add-message-property key="someData" value="1000"/>
        </message-properties-transformer>/>
        <mql:transform 
            query="from payload
                     join mule.send('vm://twitter', u.twitterId) as twitterInfo
                     select new { 
                        name = firstName + ' ' + lastName, 
                        data = someData 
                     }" />
    </flow>

You can also join data from cloud connectors or other beans inside your Mule 
configuration file. For example, let's say you define a twitter cloud connector:

(WARNING: I haven't verified this syntax for this connector, but the idea
itself is sound and should work)

    <twitter:config name="twitter" ... />
    
You can then join it in using the bean name:

    <mql:transform 
        query="from payload as u 
                 join twitter.getUserInfo(u.twitterId) as twitterInfo
                 select new { 
                    name = firstName + ' ' + lastName, 
                    tweets = twitterInfo.totalTweets 
                 }" />

You can also join data from other endpoints. For example, if you had an endpoint on
vm://twitter which retreived information about a user, you could join it in via the 
mule query context variable and the send method.	

    <flow name="join">
        <inbound-endpoint address="vm://join"
            exchange-pattern="request-response" />
        <mql:transform 
            query="from payload as u 
                     join mule.send('vm://twitter', u.twitterId) as twitterInfo
                     select new { 
                        name = firstName + ' ' + lastName, 
                        tweets = twitterInfo.totalTweets 
                     }" />
    </flow>

Implementing a custom Query Context
=================================
If you have other data sources that you want to be able to join from, use in
your where query, or in your select statement, you can support these by 
implementing a LazyQueryContext. E.g., you may want to query beans in your
Spring ApplicationContext. You can implement a SpringQueryContext like this:

	public class SpringQueryContext extends LazyQueryContext {
	    private ApplicationContext applicationContext;
	
	    public SpringQueryContext(ApplicationContext applicationContext) {
	        super();
	        this.applicationContext = applicationContext;
	    }
	
	    @Override
	    public Object load(String key) {
	        if (applicationContext.containsBean(key)) {
	            return applicationContext.getBean(key);
	        }
	        return null;
	    }
	}

(NOTE: this file is included in the com.mulesoft.mql.spring package)

Let's say that you have a bean defined in your Spring context like this:

    <bean id="userDao" class="....UserDAOImpl"/>
    
Now you can write queries against it:

	// create a context for the query
	ApplicationContext applicationContext = ...; // inject via ApplicationContextAware
	Map<String,Object> queryContext = new SpringQueryContext(applicationContext);
	
	// execute the query
	Collection<User> result = 
	    Query.execute("from userDao.users where division = 'Engineering'", queryContext);
	  
This will call getPeople() on your userManager bean. 

You could also use it to join data across different beans. E.g., this query
joins a list of divisions inside a company with a list of people in each
division.
  
    from divisionDao as division 
      join userDao.getUsersForDivision(division.id) as usersInDivision
	  select new {
	     name = division.name,
	     people = usersInDivision
	  }
	  