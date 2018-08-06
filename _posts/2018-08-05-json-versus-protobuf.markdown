---
layout: post
title:  "JSON vs Protobuf"
date:   2018-08-05 15:40:56
categories: protobuf
---

Today I'll discuss some things I've learned about Protobuf as a new user of the technology, and how it might stack up to JSON (JavaScript Object Notation) in certain scenarios. My interest in this topic began at work, when I began to brainstorm methods to speed up a Java backend service that responds in JSON to transfer data to other business units. While I also have interest in the security implications of switching to Protobuf, the focus of this particular post will be on performance.

The usage of JSON in APIs is quite ubiquitous, and for good reason. It's generally lighter than XML, it's human readable, the object is in JavaScript (the most widely written programming language at the current time), and pretty much any software developer can learn to work with it due to its simplicity. And if you're working with JSON in a backend that isn't written in Node.js there are many high quality, open-source libraries to help marshall/unmarshall your data into a localized object. 

But are there any drawbacks?

Yes, and there are certaintly counterarguments to be made to favor XML over JSON, but the whole 'XML versus JSON' thing is a very heated debate and goes outside the scope of this post. For more on that topic, maybe see [this link](https://blog.securityevaluators.com/xml-vs-json-security-risks-22e5320cf529).

I want to talk about Protobuf, which offers at least two distinct advantages over JSON in my book: nice and explicitly defined data types on fields, and superior performance within certain environments. For the purposes of this post, we'll look at a Java environment.

#### So what is this "Protobuf" thing, anyway?

Protocol buffers are a new serialization format for cross-language communication open-sourced by Google a few years ago. At time of writing Protobuf is supported in C++, Java, C#, Python, JavaScript, Objective-C, Ruby, and PHP.

>Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages. ([src](https://developers.google.com/protocol-buffers/))

So putting it in my own words, it's a mechanism to serialize and deserialize objects using a very efficient binary format that doesn't seem to waste much space, and can be sent immediately as an octet-stream through HTTP or through some other mechanism. Using both a template file and the language-specific [compiler tools](https://github.com/google/protobuf/releases) provided by Google, you can easily generate the classes necessary to create or unmarshall a Protobuf-encoded byte stream representing an object filled with data.

So as is the case with JSON, sending a Protobuf message from a Java app to a Python app, then to a Ruby app, then to a PHP app, is no problem with Protobuf. As long as the correct template is used, the applications should know how to decode the incoming bytes into a working object.

So how do we create these Protobuf templates?

Protobuf templates are files that end with a ".proto" extension, and aren't too dissimilar from .xml or .json files in that they are universal and human readable. Here's an example of a .proto file that defines a message to be sent:

```
message Customer {
  optional string first_name = 1;
  optional string last_name = 2;
  required string ssn = 3;
  optional int32 rewards_points = 4;
}
```

The snake_cased property names are assigned to integers which represent "fields" that all have to be unique. Also notice that each defined property has a rule (optional/required) as well as a data type. I haven't worked with all the data types available in Protobuf yet, but just having those explicit definitions feels good.

It also appears that in proto3 syntax, you can easily implement hash maps into your .proto files which I find incredibly powerful.

#### Setting up a test environment

I wrote an experimental program to test two different forms of an "Person" class in Java, which only contains a few attributes and an inner list of phone number objects that hold an enum as one of the properties.

So to begin the process with Protobuff, we run Google's "protoc" compiler against the following .proto template which is written by hand:

```
syntax = "proto2";

option java_package = "com.wilsontheory";

message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phones = 4;
}
```

Then we use the generated class (imported as "Person" in this block of code) to construct a new Person() in our program:

```java
Person p1 = Person.newBuilder()
        .setId(1234)
        .setName("Jane Doe")
        .setEmail("janedoe@example.com")
        .addPhones(
                Person.PhoneNumber.newBuilder()
                .setNumber("123-1234")
                .setType(Person.PhoneType.HOME)
        )
        .build();
```

Using a simple Java class with your typical "getters" and "setters", we then construct another Person (a "PersonModel") with the same attributes. This is the one that will be converted into JSON: 

```java
PersonModel p2 = new PersonModel()
        .setId(4321)
        .setName("John Doe")
        .setEmail("johndoe@example.com");
p2.addNumber(new Number("555-4321", Number.PhoneType.HOME));
```

While these two objects basically contain the same amount of information about their respective persons, the way in which we'll serialization/deserialization the objects in this set-up will be dramatically different for each object due to the differences between JSON and Protobuf. 

In the case of the Protobuf object p1, I can simply call a helper method built into the generated class to retreive the bytes.

```java
System.out.println(Arrays.toString(p1.toByteArray()));
```

This prints a byte array of length 48.

```java
[10, 8, 74, 97, 110, 101, 32, 68, 111, 101, 16, -46, 9, 26, 19, 106, 97, 110, 101, 100, 111, 101, 64, 101, 120, 97, 109, 112, 108, 101, 46, 99, 111, 109, 34, 12, 10, 8, 49, 50, 51, 45, 49, 50, 51, 52, 16, 1]
```

Now we'll convert the more conventional object into a JSON string with the [GSON library](https://github.com/google/gson), and get the bytes for this print method just to take a peek at how big it is.

```java
System.out.println(Arrays.toString(new Gson().toJson(p2).getBytes()));
```

We see more than twice the size here, at 106 bytes.

```java
[123, 34, 105, 100, 34, 58, 52, 51, 50, 49, 44, 34, 110, 97, 109, 101, 34, 58, 34, 74, 111, 104, 110, 32, 68, 111, 101, 34, 44, 34, 101, 109, 97, 105, 108, 34, 58, 34, 106, 111, 104, 110, 100, 111, 101, 64, 101, 120, 97, 109, 112, 108, 101, 46, 99, 111, 109, 34, 44, 34, 112, 104, 111, 110, 101, 115, 34, 58, 91, 123, 34, 110, 117, 109, 98, 101, 114, 34, 58, 34, 53, 53, 53, 45, 52, 51, 50, 49, 34, 44, 34, 116, 121, 112, 101, 34, 58, 34, 72, 79, 77, 69, 34, 125, 93, 125]
```

I found this size difference quite surprising, given that the generated Protobuf class is coded in nearly 2,000 lines of Java while our conventional PersonModel combined with an additional Number class are written in only ~100 lines of code. There is documentation released on how Google encodes all that information into those bytes, but I haven't had a chance to read it yet. It looks very efficient.

So now that the objects are constructed, and we see the difference in size, let's see how quickly the serialization/deserialization process happens with each method.

#### Time for some data!

To measure the speed of execution of each method, I set up a simple demonstration where both methods would be tested a predetermined number of times (by the TEST_ITERATIONS constant, set to 1000 in my application) in sequence within a single thread. In each iteration, the objects are turned into their serialized formats (byte array for Protobuf and String for JSON), and subsequently consumed by helper methods that deserialize the object and retrieve a property from it -- in this case the email address of the person.

```java
long start1 = System.nanoTime();
for (int n=1; n<TEST_ITERATIONS; n++){
    try {
        getEmailFromSerializedProtobufPerson(p1.toByteArray());
    } catch (Exception e) {
        e.printStackTrace();
    }
}
long time1 = (System.nanoTime() - start1) / 1000000;

long start2 = System.nanoTime();
for (int n=1; n<TEST_ITERATIONS; n++){
    try {
        getEmailFromSerializedJsonPerson(new Gson().toJson(p2));
    } catch (Exception e){
        e.printStackTrace();
    }
}
long time2 = (System.nanoTime() - start2) / 1000000;


```

Note that System.nanoTime() does everything in nanoseconds which I find difficult to comprehend, thus the conversion into milliseconds. And here are the deserialization helper methods, turning the byte arrays and strings into usable objects.

```java
public static String getEmailFromSerializedProtobufPerson(byte[] data) throws IOException {
    return Person.parseFrom(data).getEmail();
}

public static String getEmailFromSerializedJsonPerson(String data) {
    return new Gson().fromJson(data, PersonModel.class).getEmail();
}
```

After all the iterations are done, we take a look at the data.

```java
System.out.println("EXPERIMENT 1: Person deserialization");
System.out.println("protobuf time to completion (ms): " + time1);
System.out.println("json time to completion (ms): " + time2);
long diff = time2 - time1;
System.out.println("protobuf cumulatively faster by : " + diff + "ms, ratio: " + (float) time2 / time1);
```

Which gave me, in my most recent run:

```
EXPERIMENT 1: Person deserialization
protobuf time to completion (ms): 73
json time to completion (ms): 728
protobuf cumulatively faster by : 655ms, ratio: 9.972603
```

Naturally, there's variance with the timing each time you run the program, but I've generally seen ratios in the range of 6-11 implying that Protobuf serialization/deserialization can regularly outperform JSON by a factor of 10 without factoring in network latency. But factoring in the decreased size of the Protobuf object, it makes sense that it would move significantly faster over the wire compared to a JSON object. 

The conclusion is that in this very specific setup, Protobuf is better at least in terms of performance. But here are some reasons it might not be worth it:

* your API services are fast enough
* the bottleneck to the performance of your APIs lies elsewhere in the code
* you or your team don't have the capacity to switch
* you have automated components of your system that utilize JSON format
* you have JSON validation that would be difficult to refactor
* you use MEAN stack and enjoy the convenience of having JavaScript everywhere

In general, I'm impressed with Protobuf and find it a very interesting alternative to XML and JSON. When I have some more time I plan to test the relative performance of this serialization format with larger objects, to find an accurate way to include network latency into the test, and to include XML in these benchmark tests as well.

If you want to run the experiment from above on your own, tweak it, or improve it, please [feel free](https://github.com/wilsontheory/deserialization_experiment).



