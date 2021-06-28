## ANSA.DataGateHelper namespace

### Classes 

#### DGFileReader 

This class uses the DataGate API to read any DataGate file. 

[See this report for a simple AVR example that uses DGFileReader.](https://github.com/ASNA/avr-DGReadFile-example)
[See this repo for an a more complext AVR example that uses the DGFileReader to export a DG file to CSV.](https://github.com/ASNA/avr-version-of-export-dg-to-csv)

## AVR class to read any file -- with an event

This project provides a class named `DGFileReader` that lets you read any DataGate physical or logical file. This is a more flexible version [of the project provided here.](https://github.com/ASNA/generic-file-reader-simple). This project uses a custom event to provide flexible processing for each row read. 

### The use case

The primary use case of this version of `DGFileReader` is enabling AVR to open and read that is unknown to AVR at compile-time. Files are usually known at compile-time and references are backed into your AVR program with `DclDiskFile` objects for each file needed. 

Consider the need to do a complex query on the IBM i and get that data back into AVR, perhaps to export that data to Excel or populate a grid or a list. 
Lately, I've been using AVR to call RPG programs with AVR that do interesting things with SQL and write the results to QTEMP. This output file and its structure are known (and in fact probably don't even exist) to AVR at compile-time; you can't define them with a `DclDiskFile` at compile-time. 

It's amazing how quickly AVR can call an RPG program to use SQL to write results to a work file -- and then open that work file. The process sounds a little convolute, but AVR can make an RPG program call in a matter of milliseconds and this process is very effective.

At first, I resisted writing the SQL results to a temporary file, for two reasons:

1. I thought it would impede program performance to call the RPG program to create a temporary work file and then immediately open and read that file with AVR. I was wrong about this. 
2. It was cumbersome to create a copy of the work file to have it available at AVR compile-time. Every time I changed the SQL statement, I had to recreate a static copy of the file. 

My early passes used a data structure array in the RPG program's parameter list to get SQL-fetched data back to the AVR program. That worked well, but imposed imposed lots of coding ceremony and friction in the both the calling AVR program and the ILE RPG called program. I needed something simpler. 

Because the "program call/write a work file/read the work file" workflow doesn't impede performance and the `DGFileReader` class makes it easy to open and process dynamically, together they provided the simpler something I needed.

Beyond working with files unknown at compile-time, `DGFileReader` may also be of value to files that you do know about at compile-time. For example, let's say you have a small file of order types and you need to put fields from that file into a drop-down box. `DGFileReader` is great for little things like that. 

Anytime you need to read a file quickly, of known structure or not (but especially of unknown structure), consider using `DGFileReader.`


Singleton DB pattern
Record blocking 
AVR versus API performance 

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

There is a little bonus feature in `DGFileReader` that might be handy in some situations. By default, its `ReadEntireFile` loops over the file and reports each row read through its `AfterRowRead` event and does not accumulate the rows read. That way, you can read a very large file with minimal memory impact; by default there is only a single row in memory at any one time. 

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

Other things to discuss:
* Two DG connections (AVR and API)
* Blocking factor
* Event args