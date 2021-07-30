

大洲: $geoip_city_continent_code
国家: $geoip_country_code, $geoip_country_code3, $geoip_country_name
城市: $geoip_region_name, $geoip_city
经纬: $geoip_longitude, $geoip_latitude \n


常用指令总结
1.geoip_country

（1）$geoip_country_code    双字符国家代码，比如 “RU”，“US”。
（2）$geoip_country_code3   三字符国家代码，比如 “RUS”，“USA”。
（3）$geoip_country_name    国家名称，比如 “Russian Federation”，“United States”。
（4）$geoip_city_continent_code  属于全球哪个洲

2.geoip_city

（1）$geoip_region_name  洲或省名称 如： 浙江
（2）$geoip_region       国家行政区名（行政区、直辖区、州、省、联邦管辖区，诸如此类），比如 “Moscow City”，“DC”
（3）$geoip_city         城市名称，如： “Moscow”，“Washington”
（4）$geoip_postal_code  邮编
（5）$geoip_longitude    经度
（6）$geoip_latitude     维度



3.geoip_proxy address | CIDR;
默认值: —
上下文: http
这个指令出现在版本 1.3.0 和 1.2.1.
定义可信地址。 如果请求来自可信地址，nginx将使用其“X-Forwarded-For”头来获得地址。


4.geoip_proxy_recursive on | off;
默认值:
geoip_proxy_recursive off;
上下文: http

curl -H 'X-Forwarded-For: 221.218.143.178' geoip.xiaoshuai.com


server {
    listen 80;
    server_name 75.125.197.200;
    root html;
    index   index.html index.htm;
    location / {
        if ($geoip_region ~ "(01|02|03|04|06|07|11|13|14|15|16|21|23|29|30|31|32|33)") {
        proxy_pass http://dianxin$request_uri;
        }
        if ($geoip_region ~ "(05|08|09|10|12|17|18|19|20|24|25|26)") {
        proxy_pass http://wangtong$request_uri;
        }
        if ($geoip_city_country_code ~ "US") {
        proxy_pass http://USA$request_uri;
        }
}


这些数字代表的是中国省份地区~~
表如下：
CN,01,"Anhui"
CN,02,"Zhejiang"
CN,03,"Jiangxi"
CN,04,"Jiangsu"
CN,05,"Jilin"
CN,06,"Qinghai"
CN,07,"Fujian"
CN,08,"Heilongjiang"
CN,09,"Henan"
CN,10,"Hebei"
CN,11,"Hunan"
CN,12,"Hubei"
CN,13,"Xinjiang"
CN,14,"Xizang"
CN,15,"Gansu"
CN,16,"Guangxi"
CN,18,"Guizhou"
CN,19,"Liaoning"
CN,20,"Nei Mongol"
CN,21,"Ningxia"
CN,22,"Beijing"
CN,23,"Shanghai"
CN,24,"Shanxi"
CN,25,"Shandong"
CN,26,"Shaanxi"
CN,28,"Tianjin"
CN,29,"Yunnan"
CN,30,"Guangdong"
CN,31,"Hainan"
CN,32,"Sichuan"
CN,33,"Chongqing"
