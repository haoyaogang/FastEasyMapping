# FastEasyMapping

[![Build Status](https://travis-ci.org/Yalantis/FastEasyMapping.png)](https://travis-ci.org/Yalantis/FastEasyMapping)

### Note
This is fork of [EasyMapping](https://github.com/lucasmedeirosleite/EasyMapping) - flexible and easy way of JSON mapping.

## Reason to be
It turns out, that almost all popular libraries for JSON mapping SLOW. The main reason is often trips to database during lookup of existing objects. So we [decided](http://yalantis.com/blog/2014/03/17/from-json-to-core-data-fast-and-effectively/) to take already existing [flexible solution](https://github.com/lucasmedeirosleite/EasyMapping) and improve overall performance.
<p align="center" >
  <img src="https://raw.githubusercontent.com/Yalantis/FastEasyMapping/efabb88b0831c7ece88e728b9665edc4d3af5b1f/Assets/performance.png" alt="FastEasyMapping" title="FastEasyMapping">
</p>

# Installation

#### Cocoapods:
```ruby
#Podfile
platform :ios, '7.0'
pod 'FastEasyMapping', '~> 1.0'
```
or add as a static library.

## Architecture

### Mapping

* `FEMMapping`
* `<FEMProperty>`
	- `FEMAttribute`
	- `FEMRelationship`

### Deserialization _(JSON to Object)_
- `FEMDeserializer`

### Serialization _(Object to JSON)_
- `FEMSerializer`

### Advanced Deserialization
- `FEMObjectStore`
- `FEMManagedObjectStore`
- `FEMManagedObjectCache`

## Usage
### Deserialization

Nowadays `NSObject` and `NSManagedObject` mapping supported out of the box. Lets take a look on how basic mapping looks like:. For example, we have JSON:

```json
{
    "name": "Lucas",
    "user_email": "lucastoc@gmail.com",
    "car": {
        "model": "i30",
        "year": "2013"
    },
    "phones": [
        {
            "ddi": "55",
            "ddd": "85",
            "number": "1111-1111"
        },
        {
            "ddi": "55",
            "ddd": "11",
            "number": "2222-222"
        }
    ]
}
```

and corresponding [CoreData](https://www.objc.io/issues/4-core-data/core-data-overview/)-generated classes: 

```objective-c
@interface Person : NSManagedObject

@property (nonatomic, retain) NSString *name;
@property (nonatomic, retain) NSString *email;
@property (nonatomic, retain) Car *car;
@property (nonatomic, retain) NSSet *phones;

@end

@interface Car : NSManagedObject

@property (nonatomic, retain) NSString *model;
@property (nonatomic, retain) NSString *year;
@property (nonatomic, retain) Person *person;

@end

@interface Phone : NSManagedObject

@property (nonatomic, retain) NSString *ddi;
@property (nonatomic, retain) NSString *ddd;
@property (nonatomic, retain) NSString *number;
@property (nonatomic, retain) Person *person;

@end
```

In order to map _JSON to Object_ and vice versa we have to describe mapping rules:

```objective-c
@implementation Person (Mapping)

+ (FEMMapping *)defaultMapping {
	FEMMapping *mapping = [[FEMMapping alloc] initWithEntityName:@"Person"];
    [mapping addAttributesFromArray:@[@"name"]];
    [mapping addAttributesFromDictionary:@{@"email": @"user_email"}];

    [mapping addRelationshipMapping:[Car defaultMapping] forProperty:@"car" keyPath:@"car"];
    [mapping addToManyRelationshipMapping:[Person defaultMapping] forProperty:@"phones" keyPath:@"phones"];

  	return mapping;
}

@end

@implementation Car (Mapping)

+ (FEMMapping *)defaultMapping {
	FEMMapping *mapping = [[FEMMapping alloc] initWithEntityName:@"Car"];
    [mapping addAttributesFromArray:@[@"model", @"year"]];

  	return mapping;
}

@end


@implementation Phone (Mapping)

+ (FEMMapping *)defaultMapping {
    FEMMapping *mapping = [[FEMMapping alloc] initWithEntityName:@"Phone"];
    [mapping addAttributesFromArray:@[@"number", @"ddd", @"ddi"]];

    return mapping;
}

@end
```

Now we can deserialize _JSON to Object_ easily:

```objective-c
FEMMapping *mapping = [Person defaultMapping];
Person *person = [FEMDeserializer objectFromRepresentation:json mapping:mapping context:managedObjectContext];
```

Or collection of objects:

```objective-c
NSArray *persons = [FEMDeserializer collectionFromRepresentation:json mapping:mapping context:managedObjectContext];
```

Or even update object:
```objective-c
[FEMDeserializer fillObject:person fromRepresentation:json mapping:mapping];

```

### Serialization

Also we can serialize an _Object to JSON_ using mapping defined above:
```objective-c
FEMMapping *mapping = [Person defaultMapping];
Person *person = ...;
NSDictionary *json = [FEMSerializer serializeObject:person usingMapping:mapping];
```

Or collection to JSON: 
```objective-c
FEMMapping *mapping = [Person defaultMapping];
NSArray *persons = ...;
NSArray *json = [FEMSerializer serializeCollection:persons usingMapping:mapping];
```

## Mapping
### FEMAttribute
`FEMAttribute` is a core class of FEM. Briefly it is a description of relationship between the Object's `property` and the JSON's `keyPath`. Also it encapsulates knowledge of how the value needs to be mapped from _Object to JSON_ and back via blocks. 

```objective-c
typedef __nullable id (^FEMMapBlock)(id value __nonnull);

@interface FEMAttribute : NSObject <FEMProperty>

@property (nonatomic, copy, nonnull) NSString *property;
@property (nonatomic, copy, nullable) NSString *keyPath;

- (nonnull instancetype)initWithProperty:(nonnull NSString *)property keyPath:(nullable NSString *)keyPath map:(nullable FEMMapBlock)map reverseMap:(nullable FEMMapBlock)reverseMap;

- (nullable id)mapValue:(nullable id)value;
- (nullable id)reverseMapValue:(nullable id)value;

@end
```

Alongside with `property` and `keyPath` value you can pass mapping blocks that allow to describe completely custom mappings.

Examples:

#### Mapping of value with same keys and type:
```objective-c
FEMAttribute *attribute = [FEMAttribute mappingOfProperty:@"url"];
// or 
FEMAttribute *attribute = [[FEMAttribute alloc] initWithProperty:@"url" keyPath:@"url" map:NULL, reverseMap:NULL];
``` 

#### Mapping of value with different keys and same type:
```objective-c
FEMAttribute *attribute = [FEMAttribute mappingOfProperty:@"urlString" toKeyPath:@"URL"];
// or 
FEMAttribute *attribute = [[FEMAttribute alloc] initWithProperty:@"urlString" keyPath:@"URL" map:NULL, reverseMap:NULL];
``` 

#### Mapping of different types:
Quite often value type in JSON needs to be converted to more useful internal representation. For example HEX to `UIColor`, `String` to `NSURL`, `Integer` to `enum` and so on. For this purpose you can use `map` and `reverseMap` properties. For example lets describe attribute that maps `String` to [NSDate](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSDate_Class/) using [NSDateFormatter](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSDateFormatter_Class/):
```objective-c
NSDateFormatter *formatter = [[NSDateFormatter alloc] init];
[formatter setLocale:[[NSLocale alloc] initWithLocaleIdentifier:@"en_US_POSIX"]];
[formatter setTimeZone:[NSTimeZone timeZoneWithAbbreviation:@"UTC"]];
[formatter setDateFormat:@"yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"];

FEMAttribute *attribute = [[FEMAttribute alloc] initWithProperty:@"updateDate" keyPath:@"timestamp" map:^id(id value) {
	if ([value isKindOfClass:[NSString class]]) {
		return [formatter dateFromString:value];
	} 
	return nil;
} reverseMap:^id(id value) {
	return [formatter stringFromDate:value];
}];
```
First of all we've defined [NSDateFormatter](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSDateFormatter_Class/) that fits our requirements. Next step is to define Attribute instance with correct mapping. Briefly `map` block is invoked during deserialization (_JSON to Object_) while `reverseMap` is used for serialization process. Both are quite stratforward with but with few gotchas: 

- `map` can receive `NSNull` instance. This is a valid case for `null` value in JSON.
- `map` won't be invoked for missing keys. Therefore if JSON doesn't contain `keyPath` specified by your attribute, reverse mapping not called.
- you can return from `map` either `nil` or `NSNull` for empty values
- `reverseMap` invoked only when `property` contains non-nil value.
- you can return from `reverseMap` either `nil` or `NSNull`. Both will produce `{"keyPath": null}`

#### Adding attribute to FEMMapping
There are several shortcuts that allows you to add attributes easier to the mapping itself:
##### Excplicitly
```objective-c
FEMMapping *mapping = [[FEMMapping alloc] initWithObjectClass:[Person class]];
FEMAttribute *attribute = [FEMAttribute mappingOfProperty:@"url"];
[mapping addAttribute:attribute];
```

##### Implicitly
```objective-c
FEMMapping *mapping = [[FEMMapping alloc] initWithObjectClass:[Person class]];
[mapping addAttributeWithProperty:@"property" keyPath:@"keyPath"];
```

##### As Dictionary
```objective-c
FEMMapping *mapping = [[FEMMapping alloc] initWithObjectClass:[Person class]];
[mapping addAttributesFromDictionary:@{@"property": @"keyPath"}];
```

##### As Array
Useful when the `property` is equal to the `keyPath`:
```objective-c
FEMMapping *mapping = [[FEMMapping alloc] initWithObjectClass:[Person class]];
[mapping addAttributesFromArray:@[@"propertyAndKeyPathAreTheSame"]];
```

### FEMRelationship
`FEMRelationship` is a class that describes relationship between two `FEMMapping` instances. 
```objective-c
@interface FEMRelationship

@property (nonatomic, copy, nonnull) NSString *property;
@property (nonatomic, copy, nullable) NSString *keyPath;

@property (nonatomic, strong, nonnull) FEMMapping *mapping;
@property (nonatomic, getter=isToMany) BOOL toMany;

@property (nonatomic) BOOL weak;
@property (nonatomic, copy, nonnull) FEMAssignmentPolicy assignmentPolicy;

@end
```

Relationship also bound to a `property` and `keyPath`. Obviously it has a reference to Object's `FEMMapping` and flag that indicates whethere it is a to-many relationship. Moreover it allows you to specify assignment policy and "weakifying" behaviour of the relationship.

Example: 

```objective-c
FEMMapping *childMapping = ...;

FEMRelationship *childRelationship = [[FEMRelationship alloc] initWithProperty:@"parentProperty" keyPath:@"jsonKeyPath" mapping:childMapping];
childRelationship.toMany = YES;
```

#### Assignment policy
Assignment policy describes how deserialized relationship value should be assigned to a property. FEM supports 5 policies out of the box:

- `FEMAssignmentPolicyAssign` - replace Old property's value by New. Designed for to-one and to-many relationship. Default policy.
- `FEMAssignmentPolicyObjectMerge` - assigns New relationship value unless it is `nil`. Designed for to-one relationship.
- `FEMAssignmentPolicyCollectionMerge` - merges a New and Old values of relationship. Supported collections are: [NSSet](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSSet_Class/), [NSArray](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSArray_Class/), [NSOrderedSet](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSOrderedSet_Class/) and their successors. Designed for to-many relationship.
- `FEMAssignmentPolicyObjectReplace` - replaces Old value with New by deleting Old. Designed for to-one relationship.
- `FEMAssignmentPolicyCollectionReplace` - deletes objects not presented in [union](https://en.wikipedia.org/wiki/Union_(set_theory)) of New and Old values sets. Union set is used as a New value. Supported collections are: [NSSet](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSSet_Class/), [NSArray](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/Foundation/Classes/NSArray_Class/), [NSOrderedSet](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSOrderedSet_Class/) and their successors. Designed for to-many relationship.

#### Adding relationship to FEMMapping

##### Excplicitly
```objective-c
FEMMapping *mapping = [[FEMMapping alloc] initWithObjectClass:[Person class]];
FEMMapping *carMapping = [[FEMMapping alloc] initWithObjectClass:[Car class]];

FEMRelationship *carRelationship = [[FEMRelationship alloc] initWithProperty:@"car" keyPath:@"car" mapping:carMapping];
[mapping addRelationship:carRelationship];
```

##### Implicitly
```objective-c
FEMMapping *mapping = [[FEMMapping alloc] initWithObjectClass:[Person class]];
FEMMapping *phoneMapping = [[FEMMapping alloc] initWithObjectClass:[Phone class]];

[mapping addToManyRelationshipMapping:phoneMapping property:@"phones" keyPath:@"phones"];
```

### FEMMapping
Generally `FEMMapping` is a class that describes mapping for `NSObject` or `NSManagedObject` by encapsulating set of attributes and relationships. In addition to this it defines possibilities for objects uniquing (supported by CoreData only).

The only difference between `NSObject` and `NSManagedObject` is in `init` methods:

##### NSObject
```objective-c
FEMMapping *objectMapping = [[FEMMapping alloc] initWithObjectClass:[CustomNSObjectSuccessor class]];
```

##### NSManagedObject
```objective-c
FEMMapping *managedObjectMapping = [[FEMMapping alloc] initWithEntityName:@"EntityName"];
```

#### Root Path
Sometimes desired JSON is nested by a keyPath. In this case you can use `rootPath` property. Lets modify Person JSON by nesting Person representation:
```json
{
	result: {
		"name": "Lucas",
    	"user_email": "lucastoc@gmail.com",
    	"car": {
        	"model": "i30",
        	"year": "2013"
    	}
	}
}
```

Mapping will looks like:
```objective-c
@implementation Person (Mapping)

+ (FEMMapping *)defaultMapping {
	FEMMapping *mapping = [[FEMMapping alloc] initWithEntityName:@"Person"];
	mapping.rootPath = @"result";

    [mapping addAttributesFromArray:@[@"name"]];
    [mapping addAttributesFromDictionary:@{@"email": @"user_email"}];
    [mapping addRelationshipMapping:[Car defaultMapping] forProperty:@"car" keyPath:@"car"];
  
  	return mapping;
}

@end
```

> IMPORTANT: `FEMMapping.rootPath` is ignore during relationship mapping. Use `FEMRelationship.keyPath` instead!

### Uniquing
It is a common case when you're deserializing JSON into CoreData and don't want to duplicate data in your database. This can be easily achieved by utilizing `FEMMapping.primaryKey`. It informs `FEMDeserializer` to track primary keys and avoid data copying. For example lets make Person's `email` a primary key attribute: 
```objective-c
@implementation Person (Mapping)

+ (FEMMapping *)defaultMapping {
	FEMMapping *mapping = [[FEMMapping alloc] initWithEntityName:@"Person"];
    mapping.primaryKey = @"email";
    [mapping addAttributesFromArray:@[@"name"]];
    [mapping addAttributesFromDictionary:@{@"email": @"user_email"}];

    [mapping addRelationshipMapping:[Car defaultMapping] forProperty:@"car" keyPath:@"car"];
  
  	return mapping;
}

@end
```

> We recommend to index your primary key in datamodel to speedup keys lookup. Supported values for primary keys are Strings and Integers.

Starting from second import `FEMDeserializer` will update existing `Person`. 

### Relationship bindings by PK
Sometimes object representation contains a relationship described by a PK of the target entity:
```
{
	"result": {
		"id": 314
		"title": "https://github.com"
		"category": 4
	}
}
```

As you can see from JSON we have two objects: `Website` and `Category`. Whereas `Website` can be imported easily, there is an external reference to a `Category` represented by its primary key `id`. Can we bind website to the corresponding category? Yep! We just need to treat Website's representation as a Category:

First of all lets declare our classes: 
```objective-c
@interface Website: NSManagedObject

@property (nonatomic, strong) NSNumber *identifier;
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) Category *category;

@end

@interface Category: NSManagedObject

@property (nonatomic, strong) NSNumber *identifier;
@property (nonatomic, strong) NSString *title;
@property (nonatomic, strong) NSSet *websites

@end
```

Now it is time to define mapping for `Website`:

```objective-c
@implementation Website (Mapping)

+ (FEMMapping *)defaultMapping {
	FEMMapping *mapping = [[FEMMapping alloc] initWithEntityName:@"Website"];
	mapping.primaryKey = @"identifier";
	[mapping addAttributesFromDictionary:@{@"identifier": @"id", @"title": @"title"}];

	FEMMapping *categoryMapping = [[FEMMapping alloc] initWithEntityName:@"Category"];
	categoryMapping.primaryKey = @"identifier";
	[categoryMapping addAttributesFromDictionary:@{@"identifier": @"category"}];

	[mapping addRelationshipMapping:categoryMapping property:@"category" keyPath:nil];

	return mapping;
}

@end

```

By specifying `nil` as a `keyPath` for the category `Website`'s representation is treated as a `Category` at the same time. In this way it is easy to bind objects that are passed by PKs (which is quite common for network). 

#### Weak relationship
In example above we have one issue: what if our database doesn't contain `Category` with `PK = 4`? By default `FEMDeserializer` creates new objects during deserialization lazily. In our case it leads to insertion of `Category` instance without any data except `identifier`. In order to prevent such inconsistencies we can set `FEMRelationship.weak` to YES:

```objective-c
@implementation Website (Mapping)

+ (FEMMapping *)defaultMapping {
	FEMMapping *mapping = [[FEMMapping alloc] initWithEntityName:@"Website"];
	mapping.primaryKey = @"identifier";
	[mapping addAttributesFromDictionary:@{@"identifier": @"id", @"title": @"title"}];

	FEMMapping *categoryMapping = [[FEMMapping alloc] initWithEntityName:@"Category"];
	categoryMapping.primaryKey = @"identifier";
	categoryMapping.weak = YES;
	[categoryMapping addAttributesFromDictionary:@{@"identifier": @"category"}];

	[mapping addRelationshipMapping:categoryMapping property:@"category" keyPath:nil];

	return mapping;
}

@end

```

As the result it'll bind `Website` to corresponding `Category` only in case the latter exists.


# Changelog

### 0.5.1
- Rename [FEMAttributeMapping](https://github.com/Yalantis/FastEasyMapping/blob/release/0.5.1/FastEasyMapping/Source/Core/Mapping/Attribute/FEMAttributeMapping.h) to [FEMAttribute](https://github.com/Yalantis/FastEasyMapping/blob/release/0.5.1/FastEasyMapping/Source/Core/Mapping/Attribute/FEMAttribute.h), [FEMRelationshipMapping](https://github.com/Yalantis/FastEasyMapping/blob/release/0.5.1/FastEasyMapping/Source/Core/Mapping/Relationship/FEMRelationshipMapping.h) to [FEMRelationship](https://github.com/Yalantis/FastEasyMapping/blob/release/0.5.1/FastEasyMapping/Source/Core/Mapping/Relationship/FEMRelationship.h)
- [Shorten FEMMapping mutation methods](https://github.com/Yalantis/FastEasyMapping/blob/release/0.5.1/FastEasyMapping/Source/Core/Mapping/FEMMapping.h#42)

### 0.4.1
- Resolves: [#19](https://github.com/Yalantis/FastEasyMapping/issues/19), [#18](https://github.com/Yalantis/FastEasyMapping/issues/18), [#16](https://github.com/Yalantis/FastEasyMapping/issues/16), [#12](https://github.com/Yalantis/FastEasyMapping/issues/12)
- Add Specs for AttributeMapping

### 0.3.7
- Added equality check before objects removal in FEMAssignmentPolicyObjectReplace
- Fixed minor issues


### 0.3.7
- Add synchronization to [FEMManagedObjectDeserializer](https://github.com/Yalantis/FastEasyMapping/blob/release/0.3.7/FastEasyMapping/Source/Core/Deserializer/FEMManagedObjectDeserializer.h#L43)
- Minor refactoring
- Fixed minor naming issues

### 0.3.3
- Update null-relationship handling in Managed Object Deserializer & Cache [handling of nil-relationship](https://github.com/Yalantis/FastEasyMapping/issues/7)

### 0.3.2
- Fix [handling of nil-relationship](https://github.com/Yalantis/FastEasyMapping/issues/7) data during deserialization
- Remove compiler warnings

### 0.3.1
- Set deployment target to 6.0
- Fix missing cache for + [FEMManagedObjectDeserializer fillObject:fromExternalRepresentation:usingMapping:]
- Update hanlding of nil relationships in assignment policies

### 0.3.0
- Add assignment policy support for FEMManagedObjectDeserializer: Assign, Merge, Replace
- Cover FEMCache by tests

### 0.2.1
- Improved types introspection by @advantis

### 0.2.0
- Renamed to FastEasyMapping

### 0.1.2
- Fixed serialization of BOOL properties on 64 bits

### 0.1.1
- Fixed caching behaviour for new objects

### 0.1
- Added managed objects cache for deserialization

# Thanks
* Special thanks to [lucasmedeirosleite](https://github.com/lucasmedeirosleite) for amazing framework.

# Extra
Read out [blogpost](http://yalantis.com/blog/from-json-to-core-data-fast-and-effectively/) about FastEasyMapping.
