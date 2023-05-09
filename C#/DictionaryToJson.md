var model = new Dictionary<string, object>()
{
    {"name", "steve" },
    {"phoneNum", 545854256 },
    { "address" , new object[] {
                       new { siteCode = "104" , siteAddr = "忠孝東路走九變" },
                       new { siteCode = "105" , siteAddr = "建國一路二段" },
                  } 
    }
};

//方法1:
var options = new System.Text.Json.JsonSerializerOptions
{
    //非 ascii 不轉 unicode
    Encoder = System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping, 
    //格式化變美觀(有空格縮排)
    WriteIndented = true,
};
string result1 = System.Text.Json.JsonSerializer.Serialize(model, options);
Console.WriteLine(result1);


//方法2:
string result2 = Newtonsoft.Json.JsonConvert.SerializeObject(model);
Console.WriteLine(result2);

//要反序列化參考 https://dotblogs.com.tw/yc421206/2022/03/27/_net6_custom_jsonconverter_deserialize_dictionary_string_object

Console.ReadKey();

