---
layout: post
title: Type-safe JSON responses in Swift
comments: true
---

>This post uses Swift 1.2 beta and the Swift 1.2 branches for both Alamofire and ObjectMapper

I'm a strong believer in type-safety. I sleep better at night if I know my JSON responses are returned as type-safe objects rather than dictionaries. It makes subtle typing bugs less likely, refactoring fields is easy and error-free and passing/storing data is much nicer. Did I mention that it is also safe? :)

I've recently started a new project where I was faced with this problem. After searching for a solution for a few days, I wasn't really satisfied with anything I found.

I finally came up with a solution that I think is easy to use, flexible and doesn't get into way too much. Of course, the effort and creativity it took is minimal compared to the libraries I used, namely [Alamofire](https://github.com/Alamofire/Alamofire) and [ObjectMapper](https://github.com/Hearst-DD/ObjectMapper). Sometimes it feels nice to stand on the shoulders of giants. 

***

## The Server Side

First let's start a simple web server that serves a static JSON. For the purposes of this post, I used Go, which is very convenient for putting something out there quickly without huge project templates or external dependencies.

{% highlight go %}
//main.go
package main

import (
  "encoding/json"
  "net/http"
)

type User struct {
  FirstName string `json:"firsName"`
  LastName string `json:"lastName"`
  Age int `json:"age"`
  AccountTypes []string `json:"accountTypes"`
}

type Crew struct {
  Members []User `json:"members"`
}

func main() {
  http.HandleFunc("/", user)
  http.ListenAndServe(":3000", nil)
}

func user(w http.ResponseWriter, r *http.Request) {
  user1 := User{"Kaan", "Dedeoglu", 26, []string{"Facebook", "Twitter", "Instagram"}}
  user2 := User{"Muratcan", "Oguz", 24, []string{"Instagram"}}
  crew := Crew{[]User{user1, user2}}

  result, err := json.Marshal(crew)
  if err != nil {
    http.Error(w, err.Error(), http.StatusInternalServerError)
    return
  }

  w.Header().Set("Content-Type", "application/json")
  w.Write(result)
}
{% endhighlight %}

I think the code is pretty self-explanatory, even if you've never coded in Go. Now let's start the server. Make sure the file is in your `GOPATH` under a folder called *staticJSON*. Run the following commands from the terminal

{% highlight bash %}
go build
./staticJSON
{% endhighlight %}

Your server should be up and running! It generates the following JSON response when we hit _http://localhost:3000_

{% highlight json %}
{
  "members": [
    {
      "firstName": "Kaan",
      "lastName": "Dedeoglu",
      "age": 26,
      "accountTypes": [
        "Facebook",
        "Twitter",
        "Instagram"
      ]
    },
    {
      "firstName": "Muratcan",
      "lastName": "Oguz",
      "age": 24,
      "accountTypes": [
        "Instagram"
      ]
    }
  ]
}
{% endhighlight %}

***

## The Swift Side

> Since I'm also targeting iOS 7, my only option to add Alamofire/ObjectMapper to the project is by manually adding the source files. This approach is fine but it loses the namespaces, so the method `request` would become `Alamofire.request` if you're adding the library as a dynamic framework.

Make sure you've imported both Alamofire and ObjectMapper into your project.

Alamofire uses the `Serializer` type for encoding response data into different formats. Here's its definition:


{% highlight swift %}
typealias Serializer = (NSURLRequest, NSHTTPURLResponse?, NSData?) -> (AnyObject?, NSError?)
{% endhighlight %}

There are a few serializers that are included in the library:

`responseDataSerializer` --- Return the data as is

`stringResponseSerializer` --- Return the data as a string

`JSONResponseSerializer` --- Return the data as a JSON object

`propertyListResponseSerializer` --- Return the data as a property list object

It is worth noting that as a client you never use these directly, instead, Alamofire comes with response methods that uses these. Here's a list straight from the documentation:

`response()`

`responseString(encoding: NSStringEncoding)`

`responseJSON(options: NSJSONReadingOptions)`

`responsePropertyList(options: NSPropertyListReadOptions)`


With this knowledge, let's get to work. Our plan is to create a new `Serializer` that will map the received JSON to our type-safe class instances. Create a file called `AlamofireExtensions.swift`

{% highlight swift %}
//AlamofireExtensions.swift
import Foundation

extension Request {
    public func responseObject<T: Mappable>(completionHandler: (NSURLRequest, NSHTTPURLResponse?, T?, NSError?) -> Void) -> Self {
        let serializer: Serializer = { (request, response, data) in
            let JSONSerializer = Request.JSONResponseSerializer(options: .AllowFragments)
            let (JSON: AnyObject?, JSONError) = JSONSerializer(request, response, data)
            if response != nil && JSON != nil {
                if let object: AnyObject = Mapper<T>().map(JSON) as? AnyObject {
                    return (object, nil)
                } else {
                    return (nil, NSError(domain: "Mapping", code: 404, userInfo: [NSLocalizedDescriptionKey:"Error mapping JSON to class"]))
                }
            } else {
                return (nil, JSONError)
            }
        }
        
        return response(serializer: serializer, completionHandler: { (request, response, object, error) in
            completionHandler(request, response, object as? T, error)
        })
    }
}
{% endhighlight %}

Here we create a generic function `responseObject` that does the magic. We create a `Serializer` that parses the data into a JSON object - and then use it to create the required instance of type **T**. 

The Mappable protocol is coming from `ObjectMapper`, let's see it in action.

Create `JSONModels.swift` and add the following code:

{% highlight swift %}
//JSONModels.swift
import Foundation

class User: Mappable {
    var firstName: String = ""
    var lastName: String = ""
    var age: Int?
    var accountTypes: [String]?
    
    required init() {
    }
    
    func mapping(map: Map) {
        firstName <= map["firstName"]
        lastName <= map["lastName"]
        age <= map["age"]
        accountTypes <= map["accountTypes"]
    }
}

class Crew: Mappable {
    var members: [User]?
    
    required init(){}
    
    func mapping(map: Map) {
        members <= map["members"]
    }
}

{% endhighlight %}

To model the responses, all we have to do is to create a class conforming to the `Mappable` protocol with the appropriate fields, and implement the method called `mapping`. It is dead simple: all you have to do is match every property name with the appropriate JSON field name.

***

Let's try it out!

Place this code in a `viewDidLoad`:

{% highlight swift %}
        
request(NSURLRequest(URL: NSURL(string: "http://localhost:3000")!)).validate().responseObject() {
    (request, response, object: Crew?, error) in
    if let crew = object, let members = crew.members {
        for user in members {
            println("\(user.firstName) \(user.lastName)")
        }
    }
}
{% endhighlight %}

Notice that we are declaring object explicitly as of type `Crew?`. This alone sets the type constraint T on the `responseObject` method and our `Serializer` maps to a Crew instance.

Run the application and you should see the following output in the console:


>Kaan Dedeoglu
>
>Muratcan Oguz

Putting a breakpoint will reveal that everything went as expected.

![Breakpoint]({{ site.url }}/assets/JSONMapping/image.png)

##Limitations

One thing that didn't work out was using `Mappable` structs instead of classes. This is due to the fact that `Serializer`'s are expected to return `AnyObject` as its result, and structs are not `AnyObject`, which is a shame since structs are the perfect value types.

