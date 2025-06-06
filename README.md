# LucaP ERP 8: Templates and protocols
#LucaP

We've been making an ERP system from scratch. The goal is to understand the craft of coding an ERP. You may never attempt what I'm doing here and write your own. Still, if you have to maintain one, you'll find the necessary skills, especially with some people adding AI-generated code to your project without cleaning it up before production. There are many decisions developers need to make, not only at the coding level but also regarding how an ERP works from the accounting and workflow levels.  

As much as I want to spend most of this newsletter covering business decisions in an MRP, I can't cover that without some foundation in how one creates an MRP. I'm building models and views that work together, regardless of the content they process.

We've been working towards that. However, take a look at this snippet from last time. From the **Chart of Accounts**, we have the **firstAccount** method

```
func firstAccount()->Account{
        return sortedAccounts().first ?? nullAccount 
    }
```

I wrote this for instructional clarity. The identifiers **firstAccount** **sortedAccounts** and **nullAccount** refer to the **Account** class. However, this will not be descriptive if I want to use the same method for the **JournalEntry** or **JournalTransaction** classes. I'll need to enter this method for every model I have. Things get messy with extensive code repetition, increasing the likelihood of mistakes and, consequently, your debugging time. 

What if I wrote this instead: 

```
func first()->Row{
        return sortedRows().first ?? blank
    }
```

if I could assign **ChartOFAccounts** as **Row** and rename **sortedAccount()** as **sortedRows()** this becmone more generic, but readable code. 

What in other programming languages are called *interfaces* are called *protocols* in  Swift. You add them to your defined objects, and they force you to add specific methods and properties to your object. Using protocols, we can solve our problem of transferring methods. 

Let's look at three examples of factory protocols. **View** is a protocol. There is one required property, **body**, that returns a View. Every view in SwiftUI must have this code:

```
struct MyView: View{
   var body: some View{
   
   }
}
```

Another common protocol is **Identifiable**. Here it is in **AccountCategory**. With the rquired variable **id**:

```
class AccountCategory: Identifiable{
    var id: Int

```  

**Identifiable** is SwiftUI's way of handling iteration with collections. You can learn more about that in [SwiftUI Essential training](https://www.linkedin.com/learning/swiftui-essential-training-18764703/the-identifiable-protocol) 

The third is **Equatable**. It ensures the **==** operator works, and you can compare two objects. Enums, structs, and basic types adopt it automatically, but classes don't. When you need a class to adopt, you have to add the **==** operator method. For example, in **Accounts**, I'd add

```
static func == (lhs: Account, rhs: Account) -> Bool {
        return
            lhs.accountNumber == rhs.accountNumber &&
            lhs.accountName == rhs.accountName &&
            lhs.accountCategory == rhs.accountCategory &&
            lhs.isActive == rhs.isActive &&
            lhs.creditTotal == rhs.creditTotal &&
            lhs.debitTotal == rhs.debitTotal
        
    }
```

This compares all the **lhs** properties to the **rhs** matching properties. If they are all equal, the two instances of **Account** are equal, returning true. 

You can make custom protocols. For LucaP, I want to use the same CRUD methods in every view, making it much easier to make a template. 

I start by declaring a protocol, just as I would a **class** or **struct**. Let's say I want a protocol that requires the variable **isActive** to determine if rows are locked for editing. I would start with  

```
protocol Activatable{
}
```

Within the protocol, I add a variable that specifies either a computed property or a stored property. Computed properties are designated after the type with **{get}**. Stored properties like **isActive** has **{get set}** For our protocol, we want a stored property

``` 
   var isActive:Bool{get set}
```

That's a simple case. Let's do a more complex protocol. I'll create a protocol to manage everything with CRUD in my app. 

```
protocol Crudable:Identifiable{
```

Protocols may also adopt other protocols, such as **Crudable** adopting **Identifiable**. When a protocol adopts another protocol, it gets access to all of its properties and methods. **Identifiable** gets me access to the type **ID**. Any class that adopts **Crudable** will also adopt **Identifiable**. 

Protocols will be using different types. To keep them all straight, if you are using the type within the protocol, you can set an associated type. For example, I set **Row** here to stand for a row object, again adopting **Identifiable**.  

```
    /// the type for the record in the table
    associatedtype Row:Identifiable
```

I can then add properties. Again, a stored property would look like this with both a get and set. 

```    
    var table:[Row]{get set}
```

Computed properties have only a get:

```
    var blank:Row{get}
```

In both cases, I'm not looking for a value yet. These are the required declarations. I'll make those values in what adopts it. 

The same is true of methods. Here are the methods we need for **Crudable**, based on previous work we've done.  

```    
// C is for Create
    func add(row:Row)-> ErrorType
// R is for Read
    func exists(id:Row.ID)-> Bool
    func find(id:Row.ID) -> (row:Row,errorType:ErrorType)
    func index(id:Row.ID)->Int!
    func firstRow()->Row
    func lastRow()->Row
    func nextRow(id:Row.ID)->Row
    func previousRow(id:Row.ID) -> Row
// U is for update
    func update(old:Row,new:Row,isActive:Bool)->ErrorType
    func update(id:Row.ID,new:Row,isActive:Bool)->ErrorType 
// D is for delete
    func remove(id:Row.ID)->ErrorType
}
```

Some methods ask for type **Row**, and some for **Row.ID**. The **ID** is the ID of the **Identifiable** Protocol. 

You'd get an error if you tried to add a method to any of these declarations. However, many of these will always be the same, and we would like to create those methods ahead of time, much like a template. To build more of a template, you use an extension to the protocol. Extensions add methods and computed properties to an object. Many of my methods will be the same for every case. As the **Row** type can be any type conforming to the protocol, I can write methods using **Row**, which substitutes for the model I'm writing with the protocol's variables. For example, **exists**. 

```
extension Crudable{
    func exists(id:Row.ID)-> Bool{
        table.contains(where:{$0.id == id})
    }
}
```

**ROW.ID**. is the row's **id** property but with a non-specific type acceptable for the property requirements of **Identifiable**. I use it like I would any other type. At adoption time, **Row** will know it's the type of **ID**. Here are some more of the functions I can add. 

```
extension Crudable{
    func exists(id:Row.ID)-> Bool{
        table.contains(where:{$0.id == id})
    }
    func index(id:Row.ID)->Int!{
        table.firstIndex(where: {$0.id == id})
    }
    func find(id:Row.ID) -> (row:Row,errorType:ErrorType){
        if let row = table.first(where:{item in item.id == id }){
            return (row,.noError)
        } else {
            return (blank,.recordNotFound)
        }
    }
    func lastRow()->Row{
        return table.last ?? blank
    }
    func firstRow()->Row{
        return table.first ?? blank
    }
    func previousRow(id:Row.ID)->Row{
        if let index = index(id:id) {
            let newIndex = index - 1
            if newIndex >= 0{
                return table[newIndex]
            } else {
                return table.last ?? blank
            }
        }
        return blank
    }
    func nextRow(id:Row.ID)->Row{
        if let index = index(id:id){
            let newIndex = index + 1
            if newIndex != table.count{
                return table[newIndex]
            } else {
                return table.first ?? blank
            }
        }
        return blank
    }
}

```

These only return a value; they do not change values. They are *immutable* methods. Extensions assume immutable methods. However, methods like **add**, **update**, and **delete** are *mutable*; they change the values in the table. I have two options to handle this. One is to pass the table as a parameter, but that would require writing **add** in the protocol like this:

```
add(row:ROW from table:[ROW])
``` 

Then write the code in the extension to match. Another option is to explicitly declare this mutable in the method with the **mutating** keyword: 

```
mutating func add(row:Row)-> ErrorType{
         if self.exists(id: row.id){
            return .recordExists
        } else {
            table.append(row)
        }
        return .noError
    }
```

We can do the same for update and delete. 

```
// U is for update
    mutating func update(oldRow:Row,newRow:Row,isActive:Bool)->ErrorType{
            if let index = table.firstIndex(where: {
                row in row.id == newRow.id}){
                if isActive{
                    table[index] = newRow
                    return .noError
                } else { // inactive account can only toggle activity
                    if isActive{
                        table[index].isActive = true
                        return .noError
                    } else {
                        return .readOnly
                    }
                }
            }
            return .recordNotFound
        } 
    
    mutating func update(id:Row.ID,newRow:Row,isActive:Bool)->ErrorType{
        if let index = index(id: id){
            if isActive{
                table[index] = newRow
                return .noError
            } else { // inactive account can only toggle activity
                if newRow.isActive{
                    table[index].isActive = true
                    return .noError
                } else {
                    return .readOnly
                 }
            }
        }
        return .recordNotFound
    }
    
    // D is for delete
    mutating func remove(id:Row.ID)->ErrorType{
        if let index = index(id: id){
            table.remove(at: index)
            return .noError
        } else {
            return .noDelete
        }
    }
```

## Adopting a protocol
Once here is a protocol, you can adopt it. Let's start with a very simple model for a row: 
```
struct TestModelRow{
    var name:String
}
```

I'm going to add three protocls to it. I'll start with **Identifiable**:

```
struct TestModelRow:Identifiable{
```

This declaration will give you an error message **Type 'TestModelRow' does not conform to protocol 'Identifiable'**

To satisfy the protocol, we add the variable **id**:

```
var id:Int
```

I'll also add **Activatable** and **Equatable**. We previously discussed the required protocol for **Equatable**; however, that is only applicable to classes. The protocol is satisfied automatically for the **==** function when adopted in enums and structs. 

For **Activatable**, there is another variable requirement, namely **isActive**. With everything in place, we have: 

```
struct TestModelRow:Identifiable,Activatable,Equatable{
    var id:Int
    var isActive:Bool = true
    var name:String    
}
```

I'll make a model for use in our app as our table. I'll use **Crudable** to do all the CRUD for me. Add the companion to **associatedType** from the protocol. I defined all my methods using **Row**. I'll tell the protocol  **Row**  is **TestModelRow** by the  **typealias** declaration.

```
class TestModel:Crudable{
    typealias Row = TestModelRow
}
```

The variables **table** and **blank** set the model in terms of **Row**, though I could use **TestModelRow** here as well. 

```
var table:[Row] = []
var blank:Row{
        TestModelRow(id: -1, isActive: false, name: "")
```

I've set up a model much easier than any I've set up before. Most of the code is hiding in the protocol's extension, available for use in the view. 

## A template view

I'll also set up a template view, which I'll call **TemplateUIView**. I'm going to build all of this in a separate app called *CRUDable*. This way, I can test independently from LucaP. I first import the *controls*, *enums*, *layoutViews*, and *style* folders from Luca P. I'll also add what we built above, the protocols **Activatable** and **Crudable** in a folder *protocols*.


Much of the code is the same as our Chart of Accounts form, though refactored to use **Row** instead of accounts generically. 

You'll, of course, add new state variables for the subviews. 

```
//-----------------State Variables for custom UI
    @State var name:String = ""
    @State var id:Int = 0
    @State var isActive = true
//----------------- End of State variables
```

There will be modifications you'll need to view as you change models. For example, in the **ChangeDisplayMode** method, **.add** might change depending on the key. Accounts have a user-generated key, but other forms, like journal transactions, have system-generated ones. User-generated keys will, at minimum, check for the uniqueness of the key. For system-generated ones, such as sequential **Int**, the next will automatically generate. My **.add** is an example of that system-generated one: 

```
case .add:                              
	let ids:[Int] = model.table.map({$0.id})
	let newID = (ids.max() ?? 0) + 1
	selectedRow = TestModelRow(id: newID,isActive: true,name:"")
	formActive = true
	keyActive = false
	message = "Add new"         
``` 
 
The new ID generates from the last ID. 

In **performAction**, I'll change the **newRow** to reflect a new instance of my row struct. 

```
let newRow = TestModelRow(id: id, isActive: isActive, name: name)
```


Finally, we add in our fields. 

```
// ----------------- code for the forms goes here
            VStack(alignment:.leading){
                LPIntField(label: "ID", value: $id, isActive: keyActive)
                LPTextField(label: "Name", contents: $name, isActive: formActive)
                LPCheckBox(label: "Active", value: $isActive, isActive: true)
           
            .padding()
            
            Text("Names").font(.title2)
            Divider()
            List(model.table){row in
                HStack{
                    Text(row.id,format:.number)
                    Text(row.name)
                    Spacer()
                    LPCheckBox(label: "Active", value: .constant(row.isActive), isActive: false)
                }
                .fontWeight(row.isActive ? .bold : .light)
                .foregroundStyle(row.isActive ? .primary : .secondary)
            }
            }
            
//--------------------- end of form code
```  

Surrounding this is our toolbars, whihc we never have to touch. 

Running all this, we get a form.


With little modification, I can re-use this view for amny of the modules we'll be writing as we move forward in the project. This template setup save us time as we build the myriad forms we'll use to build Luca ERP. 

The whole code: 

you can find the code here for both LucaP up to this point including the Protocols and the Crudable test app here:  

