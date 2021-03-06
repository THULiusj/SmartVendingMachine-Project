---
layout: post
title:  "QT Kafka Security Solution on Azure"
author: "Lit Li"
author-link: "#"
#author-image: "{{ site.baseurl }}/images/authors/photo.jpg"
date:   2017-06-06
categories: IoT
color: "blue"
#image: "{{ site.baseurl }}/images/imagename.png" #should be ~350px tall
excerpt: This article is aimed a providing China QT Kafka on Azure Security Solution Ascend+ win articles.
language: English
verticals: Hospitality
---

## China QT Kafka Security Solution on Azure ##
 
## Customer ##
ParKonect (a.k.a. QTParking)(http://qtparking.com/) is an IoT enterprise solution provider. They aim to build an IoT nerve center, to provide connectivity, integration and application services for Parking Lot.

QTParking’s innovative cloud-based IoT solution (including IoT gateway “Metis BOX”, and PaaS service “QT-Cloud”), QTParking provides OTT (Over the Top) enterprise service suite called “Intelligent Parking Management Service (IPMS)” to both property owners and government decision makers, while enabling car owners to book a parking space in advance via WeChat.

QTParking has a team of professional smart city experts, engineers, and business professionals from Fortune 500 Companies and top universities. QTParking has won over trust and backing from many leading players in the real estate business in China such as Swire Properties, China Resources, Beijing Tourism Group, Bloomage International, and worked with different SI or government partners building city-scale smart parking platform in Wuhan, Tianjin and Chongqing.
  
## Pain point ##

QTParking builds QT-Cloud as PaaS service on Azure, based on Kafka architecture and Azure IaaS, to collect and process the Metis Box message, support the customer mobile application. 

So far, QTParking’s QT-Cloud solution didn’t provide Metis Box managment function such as: devices registration, device authentication and device status check. Cause many problems existing, such as failure to send by producer, failure to get data by consumer, delay on data transmission, when Kafka is restarted, always received repeated data etc. 

These issues have seriously affected customer experience and parking service quality. QTParking’s want to build a Karfa security solution on Azure, to meet their customer experience and security demand. 

## Solution ##

QTParking want leverage Microsoft DX engagment, based on Kafka architecture to design and develop Metis Box management services, including: register, unregister, authentication, and status monitor.  So, QT-Cloud services will use these service API safely manager their Metis box,and prevent other security attacks, and reduce the data repeatly sending and realtime monitor Metis Box health status

In this Kafka Security on Azure solution, QTParking will use following Microsoft Technology 

- Kafka 
- Azure VM
- MySQL Database on Azure
- Storage


## Architecture ##
The QTParking Kafka Security on Azure solution architecture will be represented as follows:
- Update the QT-Cloud Paas Services architect to add device managment Model in API service and in Kafka mesessage design as Figure 1.


Figure 1. QT-Cloud Paas Services Architecture![QT-Cloud Paas Services Architecture](/Images/QTParking_001.PNG)

- Define Device Managment Model architecture and message flow in this model, through this model Metis Box as a IoT Gateway can send the device register request to implement it securely register and get the access Token (Reference the red message flow). Also, Metix Box can use device managment API to send it's health status data(Reference the green message flow). 

Figure 2. QT-Cloud Device Management Model Architecture![QT-Cloud Device Management Model Architecture](/Images/QTParking_002.PNG)

## Device used & Code artifacts
Microsoft's China DX Technical Evangelist team and the QTParking dev team split the engagement into three segments 
- Kafka Message Process Integration
- Device Management Message Design
- Device Management Model Development

### Kafka Message Process Integration
In this segment, QTParking team focus on process the device request from Metis Box, identify message type, if it is register message, will store to register message repositry, and will trigge Device Managemnt Model to proccess it. Kafka message process code screenshot as following:

```java
public interface IDealer {

	public void deal(ConsumerRecord<String, String> msg);
	public void dealMsgs(Vector<ConsumerRecord<String, String>> msg); 
}

public interface KafkaService {

	void sendMessage(String topicName, String message);

}

public class KafkaServiceImpl implements KafkaService {
	private static final Logger LOGGER = LoggerFactory.getLogger(KafkaServiceImpl.class);
	private KafkaProducer<String, String> inner;

	private Properties producerProperties;

	private Properties consumerProperties;

	......

	public void init() throws IOException {
		
		inner = new KafkaProducer<String, String>(producerProperties);
		KafkaConsumerService consumerThread = new KafkaConsumerService(topic, consumerProperties);
		consumerThread.start();
	}

	@Override
	public void sendMessage(String topicName,String message) {
		String key = generateKey(8);
		ProducerRecord<String, String> record = new ProducerRecord<>(topicName, key, message);
        inner.send(record,
		        new Callback() {
		            public void onCompletion(RecordMetadata metadata, Exception e) {
		                if(e != null) {
		                    LOGGER.error("kafka发送消息错误"+e);
		                  
		                } else {
		                
		                    LOGGER.info("kafka消息发送成功===>The offset of the record we just sent is: " + metadata.offset()
		                    		);
		                }
		            }
		        });
	    
	}
	......

	class KafkaConsumerService extends Thread implements IDealer {
		private final KafkaConsumer<String, String> consumer;
		private final Dispatcher dispatcher = new Dispatcher(this,poolSize,queueDrainSize);

		public KafkaConsumerService(String topic,Properties props) {
			consumer = new KafkaConsumer<>(props);
			List<String> topicList = FastJSONHelper.deserializeAny(topic,new TypeReference<List<String>>(){});
			consumer.subscribe(topicList);
		}

		@Override
		public void run() {
			while(true){
				LOGGER.info("KafkaServiceImpl===============doWork方法开始==============");
				try {
					try{
						Thread.sleep(15000);
					} catch(Exception ex) {
						LOGGER.error(ex.getMessage(), ex);
					}
					long consumerRecordBefore=Calendar.getInstance().getTimeInMillis();
					ConsumerRecords<String, String> records = consumer.poll(2000);//2s
					long consumerRecordAfter=Calendar.getInstance().getTimeInMillis();
					LOGGER.info("==========从kafka抓取数据后=========总耗时:"+(consumerRecordAfter-consumerRecordBefore)+"ms 总记录数:"+records.count()+"条");
					if(records.count()>0){
						for (ConsumerRecord<String, String> record : records) {
							//dealMsgInput(record);
							LOGGER.info("KafkaServiceImpl===============放入队列的消息=============="+record.value());
							//往队列中存放信息
							dispatcher.putMsg(record);
						}
					}

				} catch (Exception e) {
					LOGGER.error("KafkaServiceImpl===============异常信息开始开始==============");
					LOGGER.error(e.getMessage());
					LOGGER.error("KafkaServiceImpl===============异常信息开始开始==============");
				}
			}

		}

		//处理单条数据
		@Override
		public void deal(ConsumerRecord<String, String> msg) {
			//处理从kafka中抓取的数据
			dealMsgInput(msg);
		}
		//处理多条数据
		@Override
		public void dealMsgs(Vector<ConsumerRecord<String, String>> msg) {
			for(int i = 0;i < msg.size();i++){
				this.deal(msg.get(i));
			}
		}
		
		private void dealMsgInput(ConsumerRecord<String, String> record) {
			try {
			LOGGER.info("Received message success: (" + record.key() + ", "
				+ record.value() + ") at offset "
				+ record.offset());

			JSONObject json = JSONObject.fromObject(record.value());
			String type = json.getString("type");
			JSONObject data = json.getJSONObject("data");
			//取出 设备code 判断是否有效
			String clientCode = "";
			if (data.containsKey("clientCode")) {
			    clientCode = data.getString("clientCode");
			}
			//获取设备端数据的id
			String dataId = "";
			if(data.containsKey("dataId")){
			    dataId = data.getString("dataId");
			}
			if (StringUtil.isEmpty(clientCode)) {
			    clientCode = "wanhao_client";
			}
			LOGGER.info("clientCode is {}", clientCode);
			......
		}
		......
	}
	......
}
```
QTParking team deploy the Kafka instance on Azure VM, and use MySQL Database on Azure as data service, the Figure 3，Figure 4 and Figure 5 are screenshot of their subscribtion managment portal.

Figure 3. QT-Cloud Kafka VM on Azure![QT-Cloud Kafka VM on Azure](/Images/QTParking_003.JPG)

Figure 4. QT-Cloud Kafka VM Console on Azure![QT-Cloud Kafka VM Console on Azure](/Images/QTParking_004.JPG)

Figure 5. QT-Cloud MySQL Database on Azure![QT-Cloud MySQL Database on Azure](/Images/QTParking_009.PNG)

### Device Management Message Design
In this segment, Microsoft's China DX Technical Evangelist team help QTParking to design the device managment message content. There are three device management message defined in here:
1.  Device Managment CMD Message from Box, such as device register message 
``` 

  "header":{
	  "deviceCode":"taiguli_s_0001",
	  "topic": "d2c_device_cmd",
	  "topicPartition": "1",
	  "signature":"deviceCode=taiguli_s_0001,topic=device_join,key=ipmstaiGuliSouth0001key201705",
	  "dataCode":"taiguli_s_0001_device_cmd_join_20170320143914567"
	  },
  "data":{ 
	  "createTime": "2017-05-03 19:36:53.546",
	  "deviceCode":"taiguli_s_0001",
	  "topic": "d2c_device_cmd",
	  "dataCode":"taiguli_s_0001_device_join_ab5678_20170320143914567","parkCode": "taiguli",
	  "cmd":"device_cmd_join"
	  }
}

```
2.  Device Managment CMD Message to Box, such as register response , device restart, device upgrade, etc. 
``` 
{
  "header":{
    "deviceCode":"taiguli_s_0001",
    "topic": "c2d_device_cmd",
	"topicPartition": "1",
	"signature":"deviceCode=taiguli_s_0001,topic=device_join,key=ipmstaiGuliSouth0001key201705",
	"dataCode":"taiguli_s_0001_device_join_ack_20170320143914567"
},
  "data":{ 
   "createTime": "2017-05-03 19:36:53.546",
   "deviceCode":"taiguli_s_0001",
   "topic": "c2d_device_cmd",
   "dataCode":"taiguli_s_0001_device_join_ab5678_20170320143914567",
   "parkCode": "taiguli",
   "cmd":"device_cmd_join_ack"
	--Command Type Send to BoX
		---device_cmd_join_ack, is device register response from cloud
		---device_cmd_reset，is resart device
		---device_cmd_restart，is restart the service
		---device_cmd_update_sw，is upgrade software on Box
		---device_cmd_alarm_mask, is turn off alert message
  }
}
```
Register Response Sample as following 
```
{
  "header":{
      "deviceCode":"taiguli_s_0001",
      "topic": "c2d_device_cmd",
      "topicPartition": "1",
      "signature":"deviceCode=taiguli_s_0001,topic=device_join,key=ipmstaiGuliSouth0001key201705"
      "dataCode":"taiguli_s_0001_device_join_ack_ab5678_20170320143914567"
  },
  "data":{ 
        "createTime": "2017-05-03 19:36:53.546",
        "deviceCode":"taiguli_s_0001",
        "topic": "c2d_device_cmd",
        "dataCode":"taiguli_s_0001_device_join_ab5678_20170320143914567",
        "parkCode": "taiguli",
        "cmd":"device_cmd_join_ack"
        "token":"kadjflsadjflkjasdlfkjas9023023",
        "partitionNum":"500"
  }
}
```

3.  Device Managment Alert Message from Box, such as device health status message
```
{
  "header":{
      "deviceCode":"taiguli_s_0001",
      "topic": "device_alarm_report",
      "topicPartition": "1",
      "signature":"deviceCode=taiguli_s_0001,topic=device_alarm_report,key=ipmstaiGuliSouth0001key201705",
      "dataCode":"taiguli_s_0001_device_alarm_report_01010001_20170320143914567"
},
  "data":{
      "createTime": "2017-05-03 19:36:53.546",
      "deviceCode":"taiguli_s_0001",
      "topic": "device_alarm_report",
      "dataCode":"taiguli_s_0001_device_join_ab5678_20170320143914567","parkCode": "taiguli",
      "alarmCode": "01010001",
      "alarmTime": "2017-05-16 19:22:55",
      "alarmSource":"device:taiguli_s_0001:db:dbname..."
  }
}
```
### Device Management Model Development
In this segment, Microsoft's China DX Technical Evangelist team and QTParking team focus on device management model devlopment. This model will process the device request from box and also process the device control repuest from managment portal. Device managment model message process code screenshot as following:
``` java
@Controller
@RequestMapping(value = "/markManagement")
public class MarkManagementController extends BaseParentHandler{

	@Resource
	private MarkManagementService markManagementService;
	......

	/**
	 * 设备查阅
	 */
	@ResponseBody
	@RequestMapping(value = "/query" , method = RequestMethod.GET)
	public ModelAndView queryMarkManagement(HttpServletRequest request,Long userId, ModelMap modelMap) throws Exception{

		Map<String, Object> map = new HashMap<String, Object>();
		......
	}

	/**
	 * 新增设备
	 */
	@RequestMapping("/addMarkManagement")
	@ResponseBody
	public Map<String, Object> addMarkManagement(HttpServletRequest request) throws Exception{

		Map<String,Object> map = new HashMap<String,Object>();
		String customercode = request.getParameter("customercode");
		String clientcode = request.getParameter("clientcode");
			String parkcode = request.getParameter("parkcode");
			String parkname = request.getParameter("parkname");
		String isavailable = request.getParameter("isavailable");
		String phone = request.getParameter("phone");
		String address = request.getParameter("address");
		......
	}

	/**
	 * 删除盒子车场关系
	 * 后台修改状态的值,前台做删除的功能
	 */
	@RequestMapping("/delMarkManagement")
	@ResponseBody
	public Map<String,Object> delMarkManagement(HttpServletRequest request){
		Map<String,Object> map = new HashMap<String,Object>();
		String id = request.getParameter("id");
		if (id == null || "".equals(id)) {
			map.put("status", "fail");
			map.put("msg", "删除失败,请重试");
			return map;
		}
		int markId = Integer.parseInt(id);
		......
	}

	//更新之前预查设备最新信息
	@RequestMapping(value = "/preEditMarkManagement" ,method = RequestMethod.GET)
	@ResponseBody
	public Map<String,Object> preEditMarkManagement(HttpServletRequest request, ModelMap modelMap){
		Map<String,Object> map = new HashMap<String,Object>();
		String id = request.getParameter("id");
		......
	}

	/**
	 * 更新盒子车场关系
	 */
	@RequestMapping(value = "/updateMarkManagement" )
	@ResponseBody
	public Map<String,Object> updateMarkManagement(HttpServletRequest request, ModelMap modelMap){
		Map<String,Object> map = new HashMap<String,Object>();
		String id = request.getParameter("id");

		String customercode = request.getParameter("preCustomerCode");
		String clientcode = request.getParameter("preClientCode");
		......
	}
	......
}


```
This model can be integrated with QT-Cloud managment portal. In the managment portal, administrator can operate the device managment task such as New Device ,Edit Device ,Set Task CMD etc. As Figure 5 and Figure 6.

Figure 6. QT-Cloud Device Managment Page ![QT-Cloud Device Managment Page](/Images/QTParking_005.PNG)

Figure 7. QT-Cloud Parking Lot Satus Page ![QT-Cloud Parking Lot Satus Page](/Images/QTParking_006.PNG)

## Opportunities going forward

Through this technical engagement, the QTParking can provide the total security solution to Parking lot users, and strengthened their confidence in using Azure as their platform. As Microsoft bizspark member, QTParking will be graduate soon, they will continue using Azure as their cloud platform and will be Microsoft EA account.   

## Great Team ##
Figure 8. QTParking Hackfest Team at Microsoft

![QTParking Hackfest Team](/Images/QTParking_007.JPG)

Figure 9. QTParking Hackfest Team at Office

![QTParking Hackfest Team](/Images/QTParking_008.JPG)


Special thanks QTParking Team, Microsoft China DX Technical Evangelist team and Audience Evanglism Team. This project team includes the following:
* William Qin -     QTParking CEO
* Jim Liu  - 	    QTParking CTO
* Peter Wang -      QTParking IoT R&D Director
* Zhaohua Wang -    QTParking R&D Director
* Lingfei Kong -    QTParking Sofeware Engineer
* Penghui Zhao -    QTParking Sofeware Engineer

* Yan Zhang - MS DX Audience Evangelism Manager
* Michael Li -		MS DX Techinical Evangelist
* Zepeng She -		MS DX Techinical Evangelist
* Lit Li -			MS DX Techinical Evangelist

