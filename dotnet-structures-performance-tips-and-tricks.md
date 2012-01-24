Recently I dug into clr structure performance and found it's rather funny. Some revelations were new to me.

At first. If you have a structure containing two boolean fields, it is five times faster than structure containing three boolean fields, but it is equal to four boolean fields one! I merely return structure from method several million times and capture elapsed time by Stopwatcher (it is common scenario when we pass structure to method or return from it)

	public struct TestStruct
	{
	    public bool Field1;
	    public bool Field2;
	    public bool Field3;
	    public bool Field4;
	 
	    public static TestStruct CreateNew()
	    {
		return default(TestStruct);
	    }
	}

	static void Main(string[] args)
	{
	    const int iterations = 50000000;
	    var sw = Stopwatch.StartNew();
	 
	    for (int i = 0; i < iterations; i++)
	    {
		var a = TestStruct.CreateNew();
	    }
	 
	    sw.Stop();
	    Console.WriteLine("elapsed {0} ms", sw.ElapsedMilliseconds);
	    Console.ReadKey();
	}
