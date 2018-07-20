
# Generic Rpc

> * 不使用avro-tools 编译协议（protocol file）生成的代码进行rpc 调用
> * 使用 http 方式
> * logicalType 使用了 avro 提供的 DateConversion对逻辑类型为date 的field 进行了处理

协议文件为report.avpr

```json
{
    "namespace": "",
    "protocol": "reportProtocol",
    "doc": "report query.",
    "types": [
        {"name":"QueryModel", "type":"record",
            "fields":[
                {"name":"cccc", "type":"string"},
                {"name":"tt", "type":"string"},
                {"name":"startTime", "type":"string"},
                {"name":"endTime", "type":"string"},
                {"name":"date", "type":{"type":"int","logicalType":"date"}}
                ]
        },
         {"name":"Report", "type":"record",
            "fields":[
                 {"name":"cccc", "type":"string"},
                {"name":"tt", "type":"string"},
                {"name":"content", "type":"string"}
                ]
        },
        {"name":"ReportsModel", "type":"record",
            "fields":[
            {"name":"reports", "type":{"type": "array", "items": "Report"}}
            ]
        }
    ],
    "messages":    {
        "queryReports":{
            "doc" : "query reports",
            "request" :[{"name":"queryModel","type":"QueryModel" }],
            "response" :"ReportsModel"
        }
    }
}
```

```java
package avro.test;

import org.apache.avro.Protocol;
import org.apache.avro.data.TimeConversions;
import org.apache.avro.generic.GenericData;
import org.apache.avro.generic.GenericRecord;
import org.apache.avro.ipc.HttpServer;
import org.apache.avro.ipc.HttpTransceiver;
import org.apache.avro.ipc.Transceiver;
import org.apache.avro.ipc.generic.GenericRequestor;
import org.apache.avro.ipc.generic.GenericResponder;
import org.joda.time.LocalDate;

import java.io.File;
import java.io.IOException;
import java.net.URL;
import java.util.ArrayList;

public class GenericRpc {

    public static void main(String[] args) throws IOException {

        URL url = GenericRpc.class.getResource("report.avpr");
        Protocol protocol = Protocol.parse(new File(url.getPath()));

        GenericData.get().addLogicalTypeConversion(new TimeConversions.DateConversion());

        HttpServer httpServer = new HttpServer(new MyResponder(), 65111);
        httpServer.start();

        GenericRecord requestData = new GenericData.Record(protocol.getType("QueryModel"));
        requestData.put("cccc", "ZBAA");
        requestData.put("tt", "SA");
        requestData.put("startTime", "2018012102");
        requestData.put("endTime", "2018012104");
        requestData.put("date", LocalDate.now());

        // 初始化请求数据
        GenericRecord request = new GenericData.Record(protocol.getMessages().get("queryReports").getRequest());
        request.put("queryModel", requestData);

        Transceiver t = new HttpTransceiver(new URL("http://127.0.0.1:65111"));
        GenericRequestor requestor = new GenericRequestor(protocol, t);

        long start = System.currentTimeMillis();
        GenericRecord result = (GenericData.Record) requestor.request("queryReports", request);

        System.out.println("返回结果："+result);
        long end = System.currentTimeMillis();
        System.out.println("请求耗时："+(end - start) + "ms");

        httpServer.close();
    }

  static class MyResponder extends GenericResponder {
        static URL url = GenericRpc.class.getResource("report.avpr");
        static Protocol protocol =null;
        static {
            try {
                protocol=Protocol.parse(new File(url.getPath()));
            }catch (Exception e){
                e.printStackTrace();
            }

        }
        public MyResponder() {
            super(protocol);
        }

        @Override
        public Object respond(Protocol.Message message, Object request) throws Exception {
            System.out.println("args[0]："+message);
            System.out.println("args[1]："+request);

            Protocol protocol=getLocal();
            GenericRecord req = (GenericRecord) request;
            GenericRecord reMessage = null;

            if (message.getName().equals("queryReports")) {
                GenericRecord msg = (GenericRecord)req.get("queryModel");
                System.out.print("接收到数据："+msg);

                LocalDate date=(LocalDate)msg.get("date");
                System.out.println(date.toString());

                //取得返回值的类型
                reMessage =  new GenericData.Record(protocol.getType("ReportsModel"));
                GenericData.Record r1=new GenericData.Record(protocol.getType("Report"));
                r1.put("cccc","ZBAA");
                r1.put("tt","SA");
                r1.put("content","11111");
                GenericData.Record r2=new GenericData.Record(protocol.getType("Report"));
                r2.put("cccc","ZSSS");
                r2.put("tt","SA");
                r2.put("content","11111");
                ArrayList<GenericData.Record> records=new ArrayList<>();
                records.add(r1);
                records.add(r2);
                reMessage.put("reports", records);
            }
            return reMessage;
        }

    }

}
```

运行结果
```
args[0]：{"doc":"query reports","request":[{"name":"queryModel","type":"QueryModel"}],"response":"ReportsModel"}
args[1]：{"queryModel": {"cccc": "ZBAA", "tt": "SA", "startTime": "2018012102", "endTime": "2018012104", "date": 2018-04-11}}
接收到数据：{"cccc": "ZBAA", "tt": "SA", "startTime": "2018012102", "endTime": "2018012104", "date": 2018-04-11}2018-04-11
返回结果：{"reports": [{"cccc": "ZBAA", "tt": "SA", "content": "11111"}, {"cccc": "ZSSS", "tt": "SA", "content": "11111"}]}
请求耗时：281ms
Disconnected from the target VM, address: '127.0.0.1:58429', transport: 'socket'

Process finished with exit code 0
```
