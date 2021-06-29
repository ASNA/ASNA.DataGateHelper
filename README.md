## ANSA.DataGateHelper namespace

### Classes 

#### DGFileReader 

This class uses the DataGate API to read any DataGate file. 

[See this repo for a simple ASNA Visual RPG (AVR) example that uses `DGFileReader`.](https://github.com/ASNA/avr-DGReadFile-example)

[See this repo for an a more complex AVR example that uses the `DGFileReader` to export a DG file to CSV.](https://github.com/ASNA/avr-version-of-export-dg-to-csv)

[See this repo for an a more a C# version of the second AVR example mentioned above CSV.](https://github.com/ASNA/cs-version-of-export-dg-to-csv)

This project provides a class named `DGFileReader` that lets you read any DataGate physical or logical file dynamically at runtime. This is a more flexible version [of the project provided here.](https://github.com/ASNA/generic-file-reader-simple). This project uses a custom event to provide flexible processing for each row read. 

You can use this class with either ASNA Visual RPG or with C#. For C# use, you'll need to manually add the ASNA.DataGate.Client and ASNA.VisualRPG.Runtime assemblies. In either case, you'll also need a DataGate client license to use this class. 

### The use case

The primary use case of `DGFileReader` is to be able to open and read a file that is unknown at compile-time. For AVR, files are usually known at compile-time and references are backed into your AVR program with `DclDiskFile` objects for each file needed. 

Consider the need to do a complex query on the IBM i and get that data back into AVR, perhaps to export that data to Excel or populate a grid or a list. 
Lately, I've been using AVR to call RPG programs with AVR that do interesting things with SQL and write the results to QTEMP. This output file and its structure are not known (and in fact probably don't even exist) to AVR at compile-time; you can't define them with a `DclDiskFile` at compile-time. 

It's amazing how quickly AVR can call an RPG program to use SQL to write results to a work file -- and then open that work file. The process sounds a little convoluted, but AVR can make an RPG program call in a matter of milliseconds and this process is very effective.

At first, I resisted writing the SQL results to a temporary file, for two reasons:

1. I thought it would impede program performance to call the RPG program to create a temporary work file and then immediately open and read that file with AVR. I was wrong about this. 
2. It was cumbersome to create a copy of the work file to have it available at AVR compile-time. Every time I changed the SQL statement, I had to recreate a static copy of the file. 

My early passes used a data structure array in the RPG program's parameter list to get SQL-fetched data back to the AVR program. That worked well, but imposed imposed lots of coding ceremony and friction in the both the calling AVR program and the ILE RPG called program. I needed something simpler. 

Because the "program call/write a work file/read the work file" workflow doesn't impede performance and the `DGFileReader` class makes it easy to open and process dynamically, together they provided the simpler something I needed.

Beyond working with files unknown at compile-time, `DGFileReader` may also be of value to files that you do know about at compile-time. For example, let's say you have a small file of order types and you need to put fields from that file into a drop-down box. `DGFileReader` is great for little things like that. 

Anytime you need to read a file quickly, of known structure or not (but especially of unknown structure), consider using `DGFileReader.`

### Look ma, a custom event

Most of this project's logic is pulled directly [from this project](https://github.com/ASNA/generic-file-reader-simple). This logic lets you dynamically open and read an entire file (without a DclDiskFile). The difference here is that this project raises an `AfterRowRead` event after each row is read. 

#### Declare an event

    // Declare `AfterRowRead` event.
    DclEvent AfterRowRead
      DclSrParm Sender Type(*Object) 
      DclSrParm e Type(AfterRowReadArgs) 

#### Declare a custom event argument class

    // Provides the `e` argument for the AfterRowRead event.
    BegClass AfterRowReadArgs Extends(System.EventArgs) Access(*Public)
        DclFld DataRow Type(System.Data.DataRow) Access(*Public)  
        DclArray FieldNames Type(*String) Rank(1) Access(*Public)
        DclFld CurrentRowCounter Type(*Integer4) Access(*Public)
        DclFld TotalRowsCounter Type(*Integer8) Access(*Public)

        BegConstructor Access(*Public) 
            DclSrParm DataRow  Type(System.Data.DataRow) 
            DclSrParm FieldNames Type(*String) Rank(1)
            DclSrParm CurrentRowCounter Type(*Integer4) 
            DclSrParm TotalRowsCounter Type(*Integer8) 
            
            *This.DataRow = DataRow
            *This.FieldNames = FieldNames
            *This.CurrentRowCounter = CurrentRowCounter
            *This.TotalRowsCounter = TotalRowsCounter  
        EndConstructor
    EndClass

#### Raise the event after each read

Notice here the `AfterRowRead` event is raised, passing it the current `*This` instance as the `Sender` argument and an instance of `AfterRowReadArgs` as the `e` argument.

    // Read a record.
    DGFile.ReadSequential(DGDS, ReadSequentialMode.Next, LockRequest.Read)
    RowCounter += 1
    // Raise AfterRowRead event.
    AfterRowRead(*This, *New AfterRowReadArgs(DGDS.ActiveRow, + 
                                                *This.FieldNames, +
                                                RowCounter, +
                                                DGFile.RecordCount))

#### In the consuming code

Note that `DGFileReader` is declared with `WithEvents(*Yes)`. This is very important. Without it, the event wouldn't be raised.

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

### Accumulating the rows in a DataTable

By default, its `ReadEntireFile` loops over the file and reports each row read through its `AfterRowRead` event and does not accumulate the rows read. That way, you can read a very large file with minimal memory impact; by default there is only a single row in memory at any one time. 

However, there might be times, for files of rational size, where you want `ReadEntireFile` to collect the entire file. For example, you might want to fill a list or a grid with the contents of a small file. To do that with `DGFileReader` set its `AccumulateRows` property to `*True` before calling `ReadEntireFile`. After that call, the entire data read is present in `DGFileReader's` `DataTable` property. The code below shows how you might use `DGFileReader` to populate a `DataGridView` in a Windows form app.

    DclFld dgh Type(DGFileReader) WithEvents(*Yes) New() 

    BegSr Form1_Load Access(*Private) Event(*this.Load)
        DclSrParm sender *Object
        DclSrParm e System.EventArgs

        // Don't auto generate DataGridView columns--use only those defined at design-time.
        datagridviewCustomers.AutoGenerateColumns = *False

        // Do accumulate rows in a DataSet.
        // BTW, why, oh why, isn't this property surfaced in the DataGridView's 
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

#### Instancing `DFFileReader`

The `DGFileReader` does not establish it's own DataGate server connection. Rather, it expects a DataGate server connection be through its constructor. This pattern of passing a DataGate connection to child class is called the [Singleton DB pattern.](https://asna.com/us/tech/kb/doc/singleton-db-pattern)

In the DataGate realm, there are two distinct database connections:

* AVR establishes connections through its `DclDB` object.
* If you are using C# with the DataGate API, the API establishes connections through the `ASNA.DataGate.Client.AdgConnection` object. 

`DGFileReader` has overloads to accept either connection type (so that it can easily be used with either AVR or C#)

> Note 

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

The `e.CurrentRowCounter` provides the one-based current row number and the `e.TotalRowsCounter` provides the number of rows in the table. Interrogating `e.CurrentCounter` provides a good way to perform first-time initialization in the AfterRowRead event handler. 

    If e.CurrentRowCounter = 1 
        // Do some initalization work.
    EndIf 

    ForEach FieldName Type(*String) Collection(e.FieldNames)
        Console.WriteLine("Field {0} has a value of {1}", +
                                    FieldName, +
                                    e.DataRow[FieldName].ToString())
    EndFor 

#### Record blocking 

`DGFileReader` opens the file it uses for read-only use with a default DataGate record blocking factor (that it calculates at runtime) Its constructor takes an optional second argument that lets you specify a custom value to use for recording blocking. 

In the constructor examples below, `nnn` is the value to use for record blocking. Setting this to a value of up to 1000 _may_ help performance. A values higher than 1000 will likely impede performance. 

> Setting the record blocking argument too high will almost certainly impede performance. You can use the `ReadMillisconds` property (explained below) to monitor the impact of changing the record blocking factor.

With C#

        this.dgfr = new ASNA.DataGateHelper.DGFileReader(apiDGDB, nnn);

With AVRL

        *This.dgfr = *New DGFileReader(DGDB, nnn)


#### Other `DGFileReader` properties

* `AccumulateRows` - a Boolean value that if true causes the records read to be collected in a System.Data.DataTable. The default is false.
* `DataTable` - the DataTable populated if `AccumulateRows` is true. 

The section, "Accumulating the rows in a DataTable" from above explains the `DGFileREader's` `AccumulateRows` and `DataTable` properties in more detail. Other properties available in `DGFileReader` after having read the file are: 

* `ReadMilliseconds` - an integer presents the time taken, in milliseconds, to read the entire file (essentially timing how long the `ReadEntireFile` method took).
* `TotalRowsCounter` - this is the same integer value as is available in the `AfterRowRead` event handler. 


#### Experimental feature: Populate a custom class

`DGFileReader` has an experimental feature that populates a custom class each time a record is read. This lets you accumulate a strongly-typed collection of custom classes. This experimental feature is an alternative to using `DGFileReader's` `AccumulateRows` and `DataTable` properties. 

Three properties enable this feature: 

* `CustomClassAssembly` - A set-only value that is the assembly that owns the custom class to populate. 
* `CustomClassName` - A set-only string value that is the fully qualified name of the custom class to populate. 
* `CustomClassInstance` - A read-only instance of the custom class with its properties populated by `DGFileReader` after each row is read. 

> Why is this experimental? I am pretty sure that it works but it hasn't been tested as thoroughly as the other features in `DGFileReader`. In fact, it was a 30 minute after-thought while writing these docs! Let me know if it works for you. So far, it is working for me. 

The intent with this example's use case is to quickly create a data source for 
a dropdown list in ASP.NET. 

Consider a `Customer` class that defines a subset of a customer file like this: 

    BegClass Customer Access(*Public)
        DclProp cmcustno    Type( *Packed ) Len( 9,0 )   Access(*Public)
        DclProp cmname      Type( *Char ) Len( 40 )      Access(*Public)
    EndClass

There are many more fields in the customer file record, but this example only needs the `cmcustno` and `cmname` values. The `Customer` class is shown below. Note that all properties are lower-case. This is important. `DGReadFile` will look up each field from a record read and assign a value to matching properties in the custom class. Its field look up is case-sensitive and assumes property names are lower case. (Technically, what is case-sensitive is the .NET reflection that dynamically assigns a value to a class property.)

> Be sure to use `DclProp` to define the class's properties. .NET reflection cannot populate fields, only properties. 

Before calling `DGFileReader's` `ReadEntireFile` method, its `Assembly` property is set to the currently executing assembly (which is the assembly that owns the custom class) and its `CustomClassName` property is set to the fully-qualified name of the custom class. 

A global collection of `Customer` is declared in the `Customers` field. After each row is read, this line in the `AfterRowRead` event handler:

    Customers.Add(*This.dgfr.CustomClassInstance *As Customer) 

adds the implicitly-created instance of `Customer` to the `Customers` collection.

After `DGFileReader's` `ReadEntireFile` method runs, these four lines of code 

    dropdownlistCustomers.DataTextField = "cmname"
    dropdownlistCustomers.DataValueField = "cmcustno"
    dropdownlistCustomers.DataSource = Customers
    dropdownlistCustomers.DataBind()

populate the ASP.NET DropdownList control.

The full code is shown below: 

    Using System.Collections.Generic 
    Using System.Reflection 
    Using ASNA.DataGateHelper

    DclNameSpace MyApp 
        
    BegClass Index Partial(*Yes) Access(*Public) Extends(System.Web.UI.Page)

        DclDB DGDB DBName("*Public/DG NET Local")

        DclFld dgfr Type(DGFileReader) WithEvents(*Yes) 
        DclFld Customers Type(List (*of Customer)) New()

        BegSr Page_Load Access(*Private) Event(*This.Load)
            DclSrParm sender Type(*Object)
            DclSrParm e Type(System.EventArgs)

            Connect DGDB 

            If (NOT Page.IsPostBack)
                PopulateCustomerInstance()
            EndIf
        EndSr

        BegSr Page_Unload Access(*Private) Event(*This.Unload)
            DclSrParm sender Type(*Object)
            DclSrParm e Type(System.EventArgs)

            Disconnect DGDB 
        EndSr

        BegSr PopulateCustomerInstance
            *This.dgfr = *New DGFileReader(DGDB, 500)

            *This.dgfr.Assembly = Assembly.GetExecutingAssembly()
            *This.dgfr.CustomClassName = "MyApp.Customer"

            *This.dgfr.ReadEntireFile("examples", "cmastnewL2")

            dropdownlistCustomers.DataTextField = "cmname"
            dropdownlistCustomers.DataValueField = "cmcustno"
            dropdownlistCustomers.DataSource = Customers
            dropdownlistCustomers.DataBind()
        EndSR 

        BegSr OnAfterRowRead Event(dgfr.AfterRowRead) 
            DclSrParm Sender Type(*Object)
            DclSrParm e Type(AfterRowReadArgs) 

            Customers.Add(*This.dgfr.CustomClassInstance *As Customer) 
        EndSr 

    EndClass

    BegClass Customer Access(*Public)
        DclProp cmcustno    Type( *Packed ) Len( 9,0 )   Access(*Public)
        DclProp cmname      Type( *Char ) Len( 40 )      Access(*Public)
    EndClass 

Like `DGFileReader's` ability to accumulate a file in a `DataTable`, use this feature carefully. A file with a million rows would suck the keys off of your keyboard as .NET tries to build a million row custom collection. 

