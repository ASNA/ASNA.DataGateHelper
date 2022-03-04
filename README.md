## ANSA.DataGateHelper namespace


>This class requires at least .NET Framework 4.7.2. Be sure your project is set to at least that .NET Framework level.

The ASNA.DataGateHelper classes:

* PagedData
* IBMiSqlPage
* IBMiCmdExec

[are documented in this repo.](https://github.com/ASNA/paged-data-class-example)


## The DGFileReader class

> [AVR and C# examples of the DGFileReader class available here.](https://github.com/ASNA/dgfilereader-cs-avr-examples)

The `DGFileReader` class uses the DataGate API to read any DataGate file from beginning to end. The primary use case of `DGFileReader` is to be able to open and read a file on demand in your AVR code without needing a compile-time `DclDiskFile.`

You can use this class with either ASNA Visual RPG or with C#. For C# use, you'll need to manually add the ASNA.DataGate.Client and ASNA.VisualRPG.Runtime assemblies. In either case, you'll also need a DataGate client license to use this class. 

In the following examples, `Tester` is a file two fields:

```
CMCustno Type(*Packed) Len(,0)
CMName   Type(*Char) Le(40)
```

The `DGFileReader` has only one public method, `ReadEntireFile`. This method opens a file for read-only access, reads it to its end, and then closes it. For each record read the `OnAfterRow` even is raised. The contents of the row just read are available as a [DataRow object](https://docs.microsoft.com/en-us/dotnet/api/system.data.datarow?view=net-6.0). Field values are available in the DataRow, but must be cast or converted to their appropriate type for most uses. 

For example, consider the need to read each record of a file and and do something with the record read. 

```
DclDB DGDB DBName('*Public/Leyland')  

DclFld dgfr Type(DGFileReader) WithEvents(*Yes)

BegSr Run Access(*Public)  
    Connect DGDB 

    dgfr = *New DGFileReader(DGDB)
    dgfr.ReadEntireFile('rp_data', 'tester') 

    Disconnect DGDB 
EndSr 

BegSr OnAfterRowRead Event(dgfr.AfterRowRead) 
    DclSrParm Sender Type(*Object)
    DclSrParm e Type(AfterRowReadArgs) 

    DclFld CustomerName Type(*String)
    DclFld CustomerNumber Type(*Integer4)

    CustomerNumber = Convert.ToInt32(e.DataRow['cmcustno'])  
    CustomerName = e.DataRow['cmname'].ToString() 

    // Do something here with the record just read. 
EndSr    
```

The file used is opened dynamically and there is no need for `DclDiskFile`. The entire file is read, beginning to end (although it wouldn't too hard to add read-by-range capabilities if you were so motivated). The `DGFileReader` was written originally to be a part of a facility to export DG files to CSV format, but has since worked well in a couple of other use cases. 


### Populating a strongly-typed collection

In the following examples, `CMastNewL2` is a file with several files, but the two we care about for this example are:

```
CMCustno Type(*Packed) Len(,0)
CMName   Type(*Char) Le(40)
```

First, you need to create a class for which you want to create a strongly-typed collection. We'll use this simple two-field class, but the class could have many fields in it. Be sure to declare them as public properties so the class works with .NET's data binding (public fields don't data data bind). 

```
BegClass Customer Access(*Public)
    DclProp CMCustNo Type(*Packed) Len(9,0) Access(*Public)
    DclProp CMName Type(*String) Access(*Public)
EndClass
```

Next, create a class as shown below. It declares the `DGFileReader` class instance and the `Customers` list global to the class. 

```
BegClass CustomerList Access(*Public) 
    DclDB DGDB DBName('*Public/Leyland')  
    
    DclFld dgfr Type(DGFileReader) WithEvents(*Yes)
    DclFld Customers Type(List(*Of Customer)) New() 

    BegSr Run Access(*Public) 
        DclFld RecordCount Type(*Integer4) 

        RecordCount = GetCustomersList() 
        Console.WriteLine("Record count: {0:#,##0}", RecordCount)

        // Show values collected. Typically at this point you'd
        // bind the Customers collection as a data source to a control.  

        ForEach c Type(Customer) Collection(Customers)
            Console.WriteLine("{0:00000} {1}", c.CMCustNo, c.CMName)
        EndFor 
    EndSr

    BegFunc GetCustomersList Type(*Integer4) 
        Connect DGDB 
        
        // Instance the reader.
        dgfr = *New DGFileReader(DGDB)    
        
        // Tell the reader the type of the class you are collecting
        // A class instance is not created if you don't provide this property.
        *This.dgfr.CustomClassType = *TypeOf(Customer)
        
        // Read the `CMastNewL2` in the `rp_data` library.
        dgfr.ReadEntireFile('rp_data', 'CMastNewL2') 

        Disconnect DGDB 
        // Leave with the number of records read. 
        LeaveSr dgfr.RecordCount
    EndFunc 
    
    BegSr OnAfterRowRead Event(dgfr.AfterRowRead) 
        DclSrParm Sender Type(*Object)
        DclSrParm e Type(AfterRowReadArgs) 

        // For each row read, add the class that the `DGFileReader` creates
        // in its `CustomClassInstance` to the `Customers` collection. 

        Customers.Add(*This.dgfr.CustomClassInstance *As Customer)
    EndSr
EndClass
```

Using the `DGFileReader` makes populating a list vastly more declarative and simple. Beware, it can get you in trouble because it does read the _entire file_ and that may try to put a million customers in the `Customers` list. 

What could the use case be for reading an entire file and putting each row in a strongly typed collection? Consider wanting to use SQL on the IBM i to populate a grid in a Web app. IBM i SQL has a great limit/offset feature that makes paging through data very simple (where `limit` is essentially the page size and `offset` is essentially the page number). An AVR program calls an ILE program and executes the following SQL (where :limit and :offset variables that AVR tracks):

```
CREATE TABLE qtemp/hdu243qwe2 AS 
(
    WITH result AS     
    (
        SELECT cmcustno, cname
        FROM exmaples/cmastnewL2
        ORDER BY cname, cmcustno
        LIMIT :limit
        OFFSET :offset
    )
SELECT * FROM result
) WITH DATA     
```

The result is a uniquely-named table created in QTEMP with a page of full of data in it, in the correct sorting sequence. Varying the `SELECT` clause can vary the columns made available. Joins and other SQL magic are also easily avaiable.

Immediately after the program call AVR uses the `DGFileReader` class to populate a strongly-typed list which is then displayed on a Web page--all in about 500 milliseconds. The technique can also use SQL LIKE clause to fine-tune row selection. 

### The `ListItemHelper` class

Along with the `DGFileReader` is a `ListItemHelper` class. This class provides a `LoadList` method that very quickly populates a [ListItemCollection](https://docs.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.listitemcollection?view=netframework-4.8) with [ListItems](https://docs.microsoft.com/en-us/dotnet/api/system.web.ui.webcontrols.listitem?view=netframework-4.8) for use with dropdown lists (in both Windows or Web apps.)

In this example, the `states` file has two fields:

```
State Type(*Char) Len(48)  (A state name)
Abbrev Type(*Char) Len(2) (A two-character state abbreviation)
```

THe following code is all it takes to populate a dropdown list with help from the `ListemItemHelper`:

```
DclDB DGDB DBName('*Public/Leyland')  

DclFld lih Type(ListItemHelper) New(*This.DGDB) 
DclFld RecordsRead Type(*Integer4) 

RecordsRead = lih.LoadList('examples', 'states', 'state', 'abbrev') 
dropdownlistStates.DataSource = lih.ListItems
```

The `LoadList` method takes four arguments:

1. Library name (`exmaples`)
2. File name (`states`)
3. Text to display field name (`state`)
4. Value field name (`abbrev`)

### Look ma, a custom event

`DGFileReader` uses raises an `AfterRowRead` event after each row is read. This event handler is where you put your code for what you want to do with each row read. This event and its implementation are declared in the `DGFileReader` class.

##### The event declaration

    // Declare `AfterRowRead` event.
    DclEvent AfterRowRead
      DclSrParm Sender Type(*Object) 
      DclSrParm e Type(AfterRowReadArgs) 

##### The event's implementation

The `AfterRowReadArg` presents data back to the event hander with information about the row just read.

    // Provides the `e` argument for the AfterRowRead event.
    BegClass AfterRowReadArgs Extends(System.EventArgs) Access(*Public)
        DclFld DataRow           Type(System.Data.DataRow) Access(*Public)  
        DclArray FieldNames      Type(*String) Rank(1) Access(*Public)
        DclFld CurrentRowCounter Type(*Integer4) Access(*Public)
        DclFld TotalRowsCounter  Type(*Integer8) Access(*Public)

        BegConstructor Access(*Public) 
            DclSrParm DataRow           Type(System.Data.DataRow) 
            DclSrParm FieldNames        Type(*String) Rank(1)
            DclSrParm CurrentRowCounter Type(*Integer4) 
            DclSrParm TotalRowsCounter  Type(*Integer8) 
            
            *This.DataRow = DataRow
            *This.FieldNames = FieldNames
            *This.CurrentRowCounter = CurrentRowCounter
            *This.TotalRowsCounter = TotalRowsCounter  
        EndConstructor
    EndClass

#### In the consuming code

`DGFileReader` must be declared in a class with `WithEvents(*Yes)`. This is very important. Without it, the `AfterRowRead` event wouldn't be raised.

```
    DclDB DGDB DBName('*Public/Cypress5166')  
    DclFld dgh Type(DGFileReader) WithEvents(*Yes) 
    
    ...

    dgh = *New DGFileReader(DGDB)

    // Read the file specified. Calling `ReadEntireFile` causes 
    // the `OnAfterRowRead` event handler to be called for 
    // each row read. 
    dgfr.ReadEntireFile('Examples', 'CMastNewL2') 

    ...

    BegSr OnAfterRowRead Event(dgfr.AfterRowRead) 
        DclSrParm Sender Type(*Object)
        DclSrParm e Type(AfterRowReadArgs) 

        // Properties passed in through the e parameter:
        //   e.DataRow -- a System.Data.DataRow representing the row read.
        //   e.FieldNames -- an array of field names in the DataRow.
        //   e.CurrentRowCounter -- The current row number. 
        //   e.TotalRowsCounter -- the total row numbers. 

        // You could do anything here you need with the incoming DataRow. 
        // In this case, we're just showing the values. 

        Console.WriteLine('{0} of {1}', e.CurrentRowCounter, e.TotalRowsCounter)
        ForEach FieldName Type(*String) Collection(e.FieldNames)
            Console.WriteLine('  {0}: {1}', FieldName, e.DataRow[FieldName])
        EndFor 
        Console.WriteLine(String.Empty)

    EndSr
EndClass
```

### BeforeFirstRead event 

There is also a BeforeFirstRead event where you might want to do something just before record reading starts. 

``` 
BegSr OnBeforeFirstRead Event(dgfr.BeforeFirstRead) 
    DclSrParm Sender Type(*Object) 
    DclSrParm e Type(EventArgs) 

    Console.WriteLine('Occurs before first read.' )
EndSr
```

### Accumulating the rows in a DataTable

By default, its `ReadEntireFile` loops over the file and reports each row read through its `AfterRowRead` event and does not accumulate the rows read. That way, you can read a very large file with minimal memory impact; by default there is only a single row in memory at any one time. 

However, there might be times, for files of rational size, where you want `ReadEntireFile` to collect the entire file. For example, you might want to fill a list or a grid with the contents of a small file. To do that with `DGFileReader` set its `AccumulateRows` property to `*True` before calling `ReadEntireFile`. After that call, the entire data read is present in `DGFileReader's` `DataTable` property. The code below shows how you might use `DGFileReader` to populate a `DataGridView` in a Windows form app.

    DclFld dgh Type(DGFileReader) WithEvents(*Yes) New() 

    BegSr Form1_Load Access(*Private) Event(*this.Load)
        DclSrParm sender *Object
        DclSrParm e System.EventArgs

        // Do accumulate rows in a DataSet.
        // design-time properties! 
        dgfr.AccumulateRows = *True 

        // Read the entire file.
        dgfr.ReadEntireFile('*Public/DG Net Local', 'Examples', 'CMastNewL2') 
        
        // Assign the DGFileReader's DataTable as the DataGridView's data source.
        datagridviewCustomers.DataSource = dgfr.DataTable 
    EndSr        

Be careful with this feature. If the file you're using has a million records in it, setting `AccumulateRows` to `*True` will set your PC on fire.

#### Achieving a fluent API

`DGFileReader's` `ReadEntireFile` is function that returns the current instance of DGFileReader. This enables a "fluent" coding style that lets you assign the DataTable to a data source in one line of code. 

Fewer lines of code isn't particularly better, and a fluent API design in this example is of only marginal use, but it's an interesting example how to create a fluent API.

Using the fluent syntax, you could do this (the line continuation is for publishing purposes):
    
    datagridviewCustomers.DataSource = + 
        dgfr.ReadEntireFile('*Public/DG Net Local', 'Examples', 'CMastNewL2').DataTable 

#### Instancing `DGFileReader`

The `DGFileReader` does not establish it's own DataGate server connection. Rather, it expects a DataGate server connection be through its constructor. This pattern of passing a DataGate connection to child class is called the [Singleton DB pattern.](https://asna.com/us/tech/kb/doc/singleton-db-pattern)

In the DataGate realm, there are two distinct database connections:

* AVR establishes connections through its `DclDB` object.
* If you are using C# with the DataGate API, the API establishes connections through the `ASNA.DataGate.Client.AdgConnection` object. 

`DGFileReader` has overloads to accept either connection type (so that it can easily be used with either AVR or C#)

To instance the `DGFileReader` with C#:

    ASNA.DataGateHelper.DGFileReader dgfr;

    public void Run() 
    {

        ASNA.DataGate.Client.AdgConnection apiDGDB = 
            new ASNA.DataGate.Client.AdgConnection("*Public/DG NET Local");
        dgfr = new ASNA.DataGateHelper.DGFileReader(apiDGDB);

        ... do work

        apiDGDB.Close();
    }        

To instance the `DGFileReader` with AVR:

    DclDB DGDB 
    DclFld dgfr Type(DGFileReader) WithEvents(*Yes) 

    BegSr Run Access(*Public)
        DGDB.DBName = "*Public/DG NET Local"
        Connect DGDB 
        *This.dgfr = *New DGFileReader(DGDB)

        ... do work

        Disconnect DGDB
    EndSr         

`DGFileReader` opens the connection passed to it if necessary, but it does not close it. Closing the connection passed to `DGFileReader` is the responsibility of the parent class. 


#### Assigning the `DGFileReader` `AfterRowRead` event handler

With C#, explicitly assign the event handler after you've instanced `DGFileReader`.

    ASNA.DataGateHelper.DGFileReader dgfr;

    public void Run() 
    {

        ASNA.DataGate.Client.AdgConnection apiDGDB = 
            new ASNA.DataGate.Client.AdgConnection("*Public/DG NET Local");
        this.dgfr = new ASNA.DataGateHelper.DGFileReader(apiDGDB);

        // Explicitly assign the event handler.
        this.dgfr.AfterRowRead += OnAfterRowRead;

        // Read all rows of the `examples/cmastnew` file.
        this.dgfr.ReadEntireFile("examples", "cmastnew");

        apiDGDB.Close();
    }        

    void OnAfterRowRead(System.Object sender, 
                        ASNA.DataGateHelper.AfterRowReadArgs e)
    {
        ... do work with values in `e`.
    }

With AVR, implicitly assign the event handler with the `Event` keyword of the event handler routine.

    DclDB DGDB 
    DclFld dgfr Type(DGFileReader) WithEvents(*Yes) 

    BegSr Run Access(*Public)
        DGDB.DBName = "*Public/DG NET Local"
        Connect DGDB 
        *This.dgfr = *New DGFileReader(DGDB)

        // Read all rows of the `examples/cmastnew` file.
        this.dgfr.ReadEntireFile("examples", "cmastnew")

        Disconnect DGDB
    EndSr         

    // Use the `Event` keyword to implicitly assign the event handler.
    BegSr OnAfterRowRead Event(dgfr.AfterRowRead) 
        DclSrParm Sender Type(*Object)
        DclSrParm e Type(AfterRowReadArgs) 

        ... do work with values in `e`.
    EndSr

#### Row data available in the `AfterRowRead` event handler 

Row data is available in the `AfterRowRead` event handler in its `e` argument. The data available is:

    e.DataRow -- a System.Data.DataRow representing the row read.
    e.FieldNames -- an array of field names in the DataRow.
    e.FieldTypes -- an array of field types in the DataRow.
    e.CurrentRowCounter -- The current row number. 
    e.TotalRowsCounter -- the total row numbers. 

##### To iterate through each field name and get each field value:

    ForEach FieldName Type(*String) Collection(e.FieldNames)
        Console.WriteLine("Field {0} has a value of {1}", +
                                    FieldName, +
                                    e.DataRow[FieldName].ToString())
    EndFor 

The field values returned through the `DataRow` argument are not strongly typed (they are `System.Objects`) and need to be cast appropriately. 

##### To see field types as you iterate through the field names

An element from the `FieldTypes` array has the same ordinal position as an element from the `FieldNames` array. That is, for the third element in the `FieldTypes` array is the data type of the third element in the `FieldNames` array.

In the 

The field types reported are .NET types. For example, `string`, `decimal`, `datetime`, etc). Field types are always presented as lower case.

    DclFld Counter Type(*Integer4)
    DclFld FieldType Type(*String)

    Counter = 0 

    // Show field names and corresponding data types on the first row only.
    If e.CurrentRowCounter = 1
        ForEach FieldName Type(*String) Collection(e.FieldNames)
                FieldType = e.FieldTypes[Counter].ToString()    
                Console.WriteLine("Field {0} is type {1}", FieldName, FieldType)
                Counter +=1 
        EndFor
    EndIf 

##### Row counter values

The `e.CurrentRowCounter` provides the one-based current row number and the `e.TotalRowsCounter` provides the number of rows in the table. It's a silly example, but if you wanted to stop reading records after the fourth record read, do this

    If e.CurrentRowCounter = 4 
        e.CancelRead = *True
    EndIf 

#### Record blocking 

`DGFileReader` opens the file it uses for read-only use with a default DataGate record blocking factor (that it calculates at runtime) Its constructor takes an optional second argument that lets you specify a custom value to use for recording blocking. 

In the constructor examples below, `nnn` is the value to use for record blocking. Setting this to a value of up to 1000 _may_ help performance. A values higher than 1000 will likely impede performance. 

> Setting the record blocking argument too high will almost certainly impede performance. Use the `ReadMillisconds` property (explained below) to monitor the impact of changing the record blocking factor.

With C#

        this.dgfr = new ASNA.DataGateHelper.DGFileReader(apiDGDB, nnn);

With AVR

        *This.dgfr = *New DGFileReader(DGDB, nnn)


#### Other `DGFileReader` properties

* `AccumulateRows` - a Boolean value that if true causes the records read to be collected in a System.Data.DataTable. The default is false.
* `DataTable` - the DataTable populated if `AccumulateRows` is true. 

The section, "Accumulating the rows in a DataTable" from above explains the `DGFileREader's` `AccumulateRows` and `DataTable` properties in more detail. Other properties available in `DGFileReader` after having read the file are: 

* `ReadMilliseconds` - an integer presents the time taken, in milliseconds, to read the entire file (essentially timing how long the `ReadEntireFile` method took).
* `TotalRowsCounter` - this is the same integer value as is available in the `AfterRowRead` event handler. 

Like `DGFileReader's` ability to accumulate a file in a `DataTable`, use this feature carefully. A file with a million rows would suck the keys off of your keyboard as .NET tries to build a million row custom collection. 

