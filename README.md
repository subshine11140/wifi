#include <SPI.h>
#include <ESP8266WiFi.h>
#include <ESP8266mDNS.h>
#include <ESP8266AVRISP.h>
#include <DHTesp.h>

#define _DEBUG  //设置是否调试，注释该行取消调试 

#ifdef _DEBUG   //判断是否处于调试模式
  #define dprint(x)   sprint(x)
  #define dprintln(x) sprintln(x)
#else
  #define dprint(x)
  #define dprintln(x)
#endif



//函数名缩写
#define pmod(a,b)   pinMode(a,b)
#define dout(a,b)   digitalWrite(a,b)
#define aout(a,b)   analogWrite(a,b)
#define din(x)      digitalRead(x)
#define ain(x)      analogRead(x)
#define sprint(x)   Serial.print(x)
#define sprintln(x) Serial.println(x) 


#define DEV_ID      "574008944"                       
#define API_KEY     "DTTAEzb9ddPNmN3yqVf6UipjDHU="    
#define SER_ADDR    "api.heclouds.com"                


#define NET_SSID    "HUAWEI P20 Pro"   
#define NET_PWD     "5334d1652e022" 
#define NET_LED     2          

#define TEMP_MAX    50  
#define TEMP_MIN    0
#define HUMI_MAX    90  
#define HUMI_MIN    20

//DHT11引脚
#define DHT_PIN     D15

//全局变量
char    json_data[200];
char    packet[400];
char    json_len[4]; 
char    temp[6];
char    humi[6];
float   var;
int dustPin=0;
float dustVal=0;
int ledPower=2;
int delayTime=280;
int delayTime2=40;
float offTime=9680;
char pm[10];
float a=(dustVal/1024)*120000*0.035;
int sensorValue; //传感器输出的数据
int sum = 0;
int vout = 2;
char UV[6];
int uv =0; //紫外线等级
DHTesp dht;


void netconnect() {
  sprint("网络信息：\r\n"
        "\t名称:"NET_SSID"\r\n"
        "\t密码:"NET_PWD"\r\n"
        "连接中，请稍后");
  WiFi.begin(NET_SSID, NET_PWD);
  while(WiFi.status() != WL_CONNECTED)
  {
    sprint('.');
    dout(NET_LED, !din(NET_LED));   //逻辑取反，因返回值为有符号数，LOW(0)按位却反(~)所得结果为-1,不为HIGH(1)
    delay(500);
  }
  sprint("\r\n连接成功");
  dout(NET_LED, LOW);
}


bool netstat() {
  if(WiFi.status() == WL_CONNECTED) {
    dout(NET_LED, LOW);
    return true;
  }
  else {
    dout(NET_LED, HIGH);
    return false;
  }
}


void mk_json(char *id, char *value, char *id2, char *value2, char *id3, char *value3, char *id4, char *value4)
{
  
  memset(json_data, 0, 200);
  strcat(json_data, "{\"datastreams\":[");

  strcat(json_data, "{\"id\":\"");
  strcat(json_data, id);
  strcat(json_data, "\",\"datapoints\":[{\"value\":");
  strcat(json_data, value);
  strcat(json_data, "}]}");

  //欲增加一次上传数据项，复制此段，更改id2及value2为指定参数
  strcat(json_data, ",{\"id\":\"");
  strcat(json_data, id2); 
  strcat(json_data, "\",\"datapoints\":[{\"value\":");
  strcat(json_data, value2); 
  strcat(json_data, "}]}");

  strcat(json_data, ",{\"id\":\"");
  strcat(json_data, id3); 
  strcat(json_data, "\",\"datapoints\":[{\"value\":");
  strcat(json_data, value3); 
  strcat(json_data, "}]}");

  strcat(json_data, ",{\"id\":\"");
  strcat(json_data, id4); 
  strcat(json_data, "\",\"datapoints\":[{\"value\":");
  strcat(json_data, value4); 
  strcat(json_data, "}]}");
  strcat(json_data, "]}");

  strcat(json_data, ",{\"id\":\"");
  strcat(json_data, id5); 
  strcat(json_data, "\",\"datapoints\":[{\"value\":");
  strcat(json_data, value5); 
  strcat(json_data, "}]}");
  strcat(json_data, "]}");
  
  itoa(strlen(json_data), json_len, 10);   
}

  
void mk_packet() {
  
  memset(packet, 0, 400);
  strcat(packet, "POST /devices/");
  strcat(packet, DEV_ID);
  strcat(packet, "/datapoints HTTP/1.1"); 
  strcat(packet, "\r\napi-key:");
  strcat(packet, API_KEY);
  strcat(packet, "\r\nHost:");
  strcat(packet, SER_ADDR);
  strcat(packet, "\r\nContent-Length:");
  strcat(packet, json_len);
  strcat(packet,"\r\n\r\n");

  
  strcat(packet, json_data);
}

void setup() 
{
  Serial.begin(115200);
  pmod(NET_LED, OUTPUT);
  dout(NET_LED, HIGH);
  pmod(DHT_PIN, INPUT);
  dht.setup(DHT_PIN, DHTesp::DHT11);
  netconnect();
  
 Serial.begin(115200);
 pinMode(ledPower,OUTPUT);
 pinMode(dustPin,INPUT);
 
 Serial.begin(115200);
 pinMode(0, INPUT);       
 
}

void loop() 
{
  WiFiClient wc;
  var = dht.getTemperature();
  if(var < TEMP_MIN || var > TEMP_MAX) 
  {
    sprint("数据错误");
    return;   
  }
  dtostrf(var, 2, 1, temp);
  var = dht.getHumidity();
  if(var < HUMI_MIN || var > HUMI_MAX)
  {
    sprint("数据错误");
    return;
  }
  dtostrf(var, 3, 1, humi);  
  
  digitalWrite(ledPower,LOW);
  delayMicroseconds(delayTime);
  dustVal=analogRead(dustPin);
  delayMicroseconds(delayTime2);
  digitalWrite(ledPower,HIGH);
  delayMicroseconds(offTime);
  dtostrf(a, 5, 3, pm);
  
  sensorValue=0;
  sum=0;
  for(int i=0;i<1024;i++)   //最简单的过滤器（filter）算法 
   {  
      sensorValue=analogRead(A0);
      sum=sensorValue+sum;
      delay(2);
   }  
   vout = sum >> 10;
   vout = vout *4980.0/1024.0;
   if(vout < 50){  
          uv = 0;
        }
        else if(vout < 227){
          uv = 1;
        }
        else if(vout < 318){
          uv = 2;
        }
        else if(vout < 408){
          uv = 3;
        }
        else if(vout < 503){
          uv = 4;
        }
        else if(vout < 606){
          uv = 5;
        }
        else if(vout < 696){
          uv = 6;
        }
        else if(vout < 795){
          uv = 7;
        }
        else if(vout < 881){
          uv = 8;
        }
        else if(vout < 976){
          uv = 9;
        }
        else if(vout < 1079){
          uv = 10;
        }
        else{
          uv = 11;
        }
  dtostrf(uv, 3, 1, UV);

  
   
  mk_json("TEMP", temp, "HUMI", humi, "PM2.5", pm, "UV", UV,);
  mk_packet();
  dprint(packet);

  if(wc.connect(SER_ADDR, 80))  
    wc.print(packet);           
  else 
    if(!netstat()) 
      netconnect();
    else 
      sprint("连接到远程主机失败");
  wc.stop();
  delay(2000);
}
