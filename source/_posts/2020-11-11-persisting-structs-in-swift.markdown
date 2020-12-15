---
layout: post
title: "Persisting Structs in Swift"
date: 2020-11-11 17:20:44 -0400
comments: true
categories: [swift]
keywords: structs,persistence
description: A technique for persisting structs (which cannot adopt the NSCoding protocol)
---

My journey through iLand recently brought me to the place whereat I needed to persist the data records being created by my application. My first thought was Core Data. But then [NSHipster](http://nshipster.com/nscoding/) helped me to realize that NSCoding would be a better choice for my situation. Great! Off I go. But darn! The NSCoding protocol is restricted to class types and I am using structs (and want them for their value semantics). What to do? After some looking around, [The Red Queen Coder](http://redqueencoder.com/property-lists-and-user-defaults-in-swift/) (TL;DR sorry Red Queen) came to my rescue. I liked the Red Queen's approach but was not very satisfied with the implementation. I would like to present the results of my tweaking and polishing.

<!-- more -->

I will start out by showing the code that will be used to accomplish the persisting of structs. Then I will show an example that uses that code.

## Implementation

### The Encodable Protocol
Types (such as structs) announce their encodability (which will allow them to be persisted) by adopting the Encodable protocol. The encode method is used to store the properties of an instance of the type into a dictionary. The initializer is used to reconstruct an instance of the type from that dictionary. The initializer is failable because there is no guarantee that the dictionary that is passed in was actually encoded by the type. The initializer's parameter is optional so as to allow cleaner code at the call site. As we will see, it is these dictionaries that are persisted.

    public protocol Encodable {
        typealias Properties = Dictionary<String, Any>
        func encode() -> Properties
        init?(_ properties: Properties?)
    }


### Helper Extensions on Array
It will often occur that we have a array of objects to be encoded.
These extensions will make the usage at the call site a bit cleaner; allowing us to write `myArray.encode()` and `myArray.decode()`.

    extension Array where Element : Encodable {
        public func encode() -> [Encodable.Properties] {
            return map{ $0.encode() }
        }
    }

    extension Array where Element == Encodable.Properties {
        public func decode<T:Encodable>(type: T.Type) -> [T] {
            return flatMap{ T($0) }
        }
    }

### Persisting to User Defaults
These top level functions will save/load arrays of Encodable objects to/from User Defaults. Note that loadFromFile's type parameter specifies the type of the objects to be instantiated from each of the Encodable.Properties (i.e Dictionaries)

    public func saveToUserDefaults<T:Encodable>(_ objects: [T], withKey key: String) {
        UserDefaults.standard.set(objects.encode(), forKey: key)
    }

    public func loadFromUserDefaults<T:Encodable>(type: T.Type, withKey key: String) -> [T]? {
        return (UserDefaults.standard.array(forKey: key) as? [Encodable.Properties])?.decode(type: T.self)
    }


### Persisting to a File
These top level functions will save/load arrays of Encodable objects to/from a file in the document directory. Note that at this point we are able to use NSKeyedArchiver and NSKeyedUnarchiver because all of the items in the dictionaries are types that are known to NSKeyedArchiver and NSKeyedUnarchiver (i.e. Int, String, Double, etc).

    public func saveToFile<T:Encodable>(_ values: [T], withName name: String) -> Bool {
        do {
            let data = NSKeyedArchiver.archivedData(withRootObject: values.encode())
            try data.write(to: getUrl(forName: name), options: .atomic)
            return true
        } catch {
            print(error)
        }
        return false
    }

    public func loadFromFile<T:Encodable>(type: T.Type, withName name: String) -> [T]? {
        do {
            let data = try Data(contentsOf: getUrl(forName: name))
            if let encoded = NSKeyedUnarchiver.unarchiveObject(with: data) as? [Encodable.Properties] {
                return encoded.decode(type: type)
            }
        } catch {
            print(error)
        }
        return nil
    }

    private func getUrl(forName: String) -> URL {
        return try! FileManager.default.url(for: .documentDirectory, in: .userDomainMask, appropriateFor: nil, create: false).appendingPathComponent(forName)
    }

## A Persistence Example

### The Encodable Type
Adopting the Encodable protocol is simple: `encode` writes the properties to a dictionary, `init` restores them from a dictionary.

    struct MyData {
        let int: Int
        let double: Double
        let string: String
        let bool: Bool
        let date: Date
        let blob: Data
    }

    extension MyData: Encodable {

        func encode() -> Properties {
            return ["int": int, "double": double, "string": string, "bool": bool, "date": date, "blob": blob, ]
        }

        init?(_ properties: Properties?) {
            guard let properties = properties else { return nil }

            if let int = properties["int"] as? Int,
               let double = properties["double"] as? Double,
               let string = properties["string"] as? String,
               let bool = properties["bool"] as? Bool,
               let date = properties["date"] as? Date,
               let blob = properties["blob"] as? Data {
                   self.int = int
                   self.double = double
                   self.string = string
                   self.bool = bool
                   self.date = date
                   self.blob = blob
            } else {
                return nil
            }
        }
    }


### Using the Persistence Functions
And, finally here is how instances of MyData can be persisted.

    let myData = [MyData(int: 1, double: 1.0, string: "One", bool: true, date: Date(), blob: Data(bytes: [1,2,3,4,5,6,7,8])),
                  MyData(int: 2, double: 2.0, string: "Two", bool: false, date: Date(), blob: Data(bytes: [9,10,11,12]))]


    print("Original:")
    myData.forEach{ print("\t", $0) }
    // Prints:
    // Original:
    //     MyData(int: 1, double: 1.0, string: "One", bool: true, date: 2017-04-04 16:04:36 +0000, blob: 8 bytes)
    //     MyData(int: 2, double: 2.0, string: "Two", bool: false, date: 2017-04-04 16:04:36 +0000, blob: 4 bytes)


    print("\nEncode => Decode:")
    myData.encode().decode(type: MyData.self).forEach{ print("\t", $0) }
    // Prints:
    // Encode => Decode:
    //     MyData(int: 1, double: 1.0, string: "One", bool: true, date: 2017-04-04 16:04:36 +0000, blob: 8 bytes)
    //     MyData(int: 2, double: 2.0, string: "Two", bool: false, date: 2017-04-04 16:04:36 +0000, blob: 4 bytes)


    print("\nUser Defaults: Save => Load")
    let key = "MyData"
    saveToUserDefaults(myData, withKey: key)
    loadFromUserDefaults(type: MyData.self, withKey: key)?.forEach{ print("\t", $0) }
    // Prints:
    // User Defaults: Save => Load
    //     MyData(int: 1, double: 1.0, string: "One", bool: true, date: 2017-04-04 16:04:36 +0000, blob: 8 bytes)
    //     MyData(int: 2, double: 2.0, string: "Two", bool: false, date: 2017-04-04 16:04:36 +0000, blob: 4 bytes)


    print("\nFile: Save => Load")
    let fileName = "MyData.dat"
    if saveToFile(myData, withName: fileName) {
        loadFromFile(type: MyData.self, withName: fileName)?.forEach{ print("\t", $0) }
    }
    else {
        print("Cannot save to \(fileName)")
    }
    // Prints:
    // File: Save => Load
    //     MyData(int: 1, double: 1.0, string: "One", bool: true, date:    2017-04-04 16:04:36 +0000, blob: 8 bytes)
    //     MyData(int: 2, double: 2.0, string: "Two", bool: false, date: 2017-04-04 16:04:36 +0000, blob: 4 bytes)


### Conclusion

This has worked great for me. I have the put the implementation into my toolbox framework. I have coded my application's data structs similarly to the MyData example. And, voila! When my application restarts it is able to recover its data.

You can download a playground that contains all of the above code [here](https://github.com/verticon/PersistingStructsInSwift)
