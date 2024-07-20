# Monitor successful logins per country with Entra ID logs

In this page, we will see how to check successful logins in a list of specific countries.

This is particularly useful to monitor red/orange countries connections from employees on any device (Laptops, phones, etc)

Why only successful logins? Because those have real value when coming from unusual countries, where failed ones would create too much noise with all bruteforce attempts.

Since there is no list of countries you can fetch from Graph or other Azure logs (as far as I could find), you need to first create a table in your query with ISo-2 country codes (used by azure) and provide full name for human readibility.

### KQL variable to have the list of countries

(sorry that it is this long, queries are below )

```
let countryCSV = datatable (countryCode: string, CountryName: string)  
[ 
"AD","Andorra", 
"AE","United Arab Emirates", 
"AF","Afghanistan", 
"AG","Antigua and Barbuda", 
"AI","Anguilla", 
"AL","Albania", 
"AM","Armenia", 
"AO","Angola", 
"AQ","Antarctica", 
"AR","Argentina", 
"AS","American Samoa", 
"AT","Austria", 
"AU","Australia", 
"AW","Aruba", 
"AX","Åland Islands", 
"AZ","Azerbaijan", 
"BA","Bosnia and Herzegovina", 
"BB","Barbados", 
"BD","Bangladesh", 
"BE","Belgium", 
"BF","BurkinaFaso", 
"BG","Bulgaria", 
"BH","Bahrain", 
"BI","Burundi", 
"BJ","Benin", 
"BL","Saint Barthélemy", 
"BM","Bermuda", 
"BN","BruneiDarussalam", 
"BO","Bolivia(PlurinationalStateof)", 
"BQ","Bonaire,SintEustatius and Saba", 
"BR","Brazil", 
"BS","Bahamas", 
"BT","Bhutan", 
"BV","Bouvet Island", 
"BW","Botswana", 
"BY","Belarus", 
"BZ","Belize", 
"CA","Canada", 
"CC","Cocos(Keeling) Islands", 
"CD",",Democratic Republic of the Congo", 
"CF","Central African Republic", 
"CG","Congo", 
"CH","Switzerland", 
"CI","Côte d'Ivoire", 
"CK","Cook Islands", 
"CL","Chile", 
"CM","Cameroon", 
"CN","China", 
"CO","Colombia", 
"CR","CostaRica", 
"CU","Cuba", 
"CV","CaboVerde", 
"CW","Curaçao", 
"CX","Christmas Island", 
"CY","Cyprus", 
"CZ","Czechia", 
"DE","Germany", 
"DJ","Djibouti", 
"DK","Denmark", 
"DM","Dominica", 
"DO","Dominican Republic", 
"DZ","Algeria", 
"EC","Ecuador", 
"EE","Estonia", 
"EG","Egypt", 
"EH","WesternSahara", 
"ER","Eritrea", 
"ES","Spain", 
"ET","Ethiopia", 
"FI","Finland", 
"FJ","Fiji", 
"FK","Falkland Islands(Malvinas)", 
"FM","Micronesia (FederatedStatesof)", 
"FO","Faroe Islands", 
"FR","France", 
"GA","Gabon", 
"GB","United Kingdom of Great Britain and Northern Ireland", 
"GD","Grenada", 
"GE","Georgia", 
"GF","French Guiana", 
"GG","Guernsey", 
"GH","Ghana", 
"GI","Gibraltar", 
"GL","Greenland", 
"GM","Gambia", 
"GN","Guinea", 
"GP","Guadeloupe", 
"GQ","Equatorial Guinea", 
"GR","Greece", 
"GS","South Georgia and the South Sandwich Islands", 
"GT","Guatemala", 
"GU","Guam", 
"GW","Guinea-Bissau", 
"GY","Guyana", 
"HK","HongKong", 
"HM","Heard Island and McDonald Islands", 
"HN","Honduras", 
"HR","Croatia", 
"HT","Haiti", 
"HU","Hungary", 
"ID","Indonesia", 
"IE","Ireland", 
"IL","Israel", 
"IM","IsleofMan", 
"IN","India", 
"IO","British IndianOcean Territory", 
"IQ","Iraq", 
"IR","Iran", 
"IS","Iceland", 
"IT","Italy", 
"JE","Jersey", 
"JM","Jamaica", 
"JO","Jordan", 
"JP","Japan", 
"KE","Kenya", 
"KG","Kyrgyzstan", 
"KH","Cambodia", 
"KI","Kiribati", 
"KM","Comoros", 
"KN","Saint Kitts and Nevis", 
"KP","North Korea", 
"KR","South Korea", 
"KW","Kuwait", 
"KY","Cayma nIslands", 
"KZ","Kazakhstan", 
"LA","Lao People's Democratic Republic", 
"LB","Lebanon", 
"LC","Saint AlertLucia", 
"LI","Liechtenstein", 
"LK","Sri Lanka", 
"LR","Liberia", 
"LS","Lesotho", 
"LT","Lithuania", 
"LU","Luxembourg", 
"LV","Latvia", 
"LY","Libya", 
"MA","Morocco", 
"MC","Monaco", 
"MD","Moldova", 
"ME","Montenegro", 
"MF","SaintMartin (Frenchpart)", 
"MG","Madagascar", 
"MH","Marshall Islands", 
"MK","North Macedonia", 
"ML","Mali", 
"MM","Myanmar", 
"MN","Mongolia", 
"MO","Macao", 
"MP","Northern Mariana Islands", 
"MQ","Martinique", 
"MR","Mauritania", 
"MS","Montserrat", 
"MT","Malta", 
"MU","Mauritius", 
"MV","Maldives", 
"MW","Malawi", 
"MX","Mexico", 
"MY","Malaysia", 
"MZ","Mozambique", 
"NA","Namibia", 
"NC","New Caledonia", 
"NE","Niger", 
"NF","Norfolk Island", 
"NG","Nigeria", 
"NI","Nicaragua", 
"NL","Netherlands", 
"NO","Norway", 
"NP","Nepal", 
"NR","Nauru", 
"NU","Niue", 
"NZ","New Zealand", 
"OM","Oman", 
"PA","Panama", 
"PE","Peru", 
"PF","French Polynesia", 
"PG","Papua New Guinea", 
"PH","Philippines", 
"PK","Pakistan", 
"PL","Poland", 
"PM","Saint Pierre and Miquelon", 
"PN","Pitcairn", 
"PR","Puerto Rico", 
"PS","Palestine", 
"PT","Portugal", 
"PW","Palau", 
"PY","Paraguay", 
"QA","Qatar", 
"RE","Réunion", 
"RO","Romania", 
"RS","Serbia", 
"RU","Russian Federation", 
"RW","Rwanda", 
"SA","Saudi Arabia", 
"SB","Solomon Islands", 
"SC","Seychelles", 
"SD","Sudan", 
"SE","Sweden", 
"SG","Singapore", 
"SH","SaintHelena, Ascension and Tristanda Cunha", 
"SI","Slovenia", 
"SJ","Svalbard and JanMayen", 
"SK","Slovakia", 
"SL","Sierra Leone", 
"SM","San Marino", 
"SN","Senegal", 
"SO","Somalia", 
"SR","Suriname", 
"SS","SouthSudan", 
"ST","Sao Tome and Principe", 
"SV","El Salvador", 
"SX","Sint Maarten(Dutchpart)", 
"SY","Syrian Arab Republic", 
"SZ","Eswatini", 
"TC","TurksandCaicosIslands", 
"TD","Chad", 
"TF","French Southern Territories", 
"TG","Togo", 
"TH","Thailand", 
"TJ","Tajikistan", 
"TK","Tokelau", 
"TL","Timor-Leste", 
"TM","Turkmenistan", 
"TN","Tunisia", 
"TO","Tonga", 
"TR","Türkiye", 
"TT","Trinidad and Tobago", 
"TV","Tuvalu", 
"TW","Taiwan", 
"TZ","Tanzania", 
"UA","Ukraine", 
"UG","Uganda", 
"UM","United States Minor Outlying Islands", 
"US","United States of America", 
"UY","Uruguay", 
"UZ","Uzbekistan", 
"VA","Vatican", 
"VC","Saint Vincent and the Grenadines", 
"VE","Venezuela", 
"VG","Virgin Islands(British)", 
"VI","Virgin Islands(U.S.)", 
"VN","VietNam", 
"VU","Vanuatu", 
"WF","Wallis and Futuna",
"WS","Samoa",
"YE","Yemen", 
"YT","Mayotte", 
"ZA","South Africa", 
"ZM","Zambia", 
"ZW","Zimbabwe" 
]; 
```

So now that you have the list of countries with their names, you need to forward these logs to a Log Analytics Workspace:
- SigninLogs
- AADNonInteractiveUserSignInLogs

Once you have all those requirements, you can play with the list by only monitoring some countries, or removing others.
Some examples:
- Monitor a specific list of countries, like red countries (for most companies, that would be North Korea, China, Russia, etc)
- Monitor all connections outside of usual countries (like outside of US, or outside of EU/EEA)

### Query for monitor connections from a list of countries

```
let countryCSV = datatable (countryCode: string, CountryName: string)  
[ 
"AD","Andorra", 
...
"ZW","Zimbabwe" 
]; 
union SigninLogs, AADNonInteractiveUserSignInLogs 
| where ResultType == 0 
| where UserType != "Guest" 
| parse LocationDetails_dynamic with * 'countryOrRegion":"' countryDynamic '",' * 
| parse LocationDetails_string with * 'countryOrRegion":"' countryString '",' * 
| parse LocationDetails_string with * 'city":"' cityString '",'* 
| parse LocationDetails_string with * 'state":"' stateString '",' * 
| parse DeviceDetail_string with * 'displayName":"' deviceName '",' * 
| parse DeviceDetail_string with * 'browser":"' browser '",' * 
| where countryString in ("CN","KP","RU") 
| join countryCSV on $left.countryString == $right.countryCode 
| project TimeGenerated, Identity, UserPrincipalName, countryString, CountryName, IPAddress, cityString, stateString, deviceName, browser 
| distinct Identity, UserPrincipalName, countryString, CountryName, cityString, stateString, TimeGenerated, deviceName, browser 
```

### Query to monitor all connections outside of a country list

```
let countryCSV = datatable (countryCode: string, CountryName: string)  
[ 
"AD","Andorra", 
...
"ZW","Zimbabwe" 
]; 
union SigninLogs, AADNonInteractiveUserSignInLogs 
| where ResultType == 0 
| where UserType != "Guest" 
| parse LocationDetails_dynamic with * 'countryOrRegion":"' countryDynamic '",' * 
| parse LocationDetails_string with * 'countryOrRegion":"' countryString '",' * 
| parse LocationDetails_string with * 'city":"' cityString '",'* 
| parse LocationDetails_string with * 'state":"' stateString '",' * 
| parse DeviceDetail_string with * 'displayName":"' deviceName '",' * 
| parse DeviceDetail_string with * 'browser":"' browser '",' * 
| where countryString !in ("AD","AT","BE","CZ","BG","CH","CY","DE","DK","ES","EE","FO","FI","FR","GB","GG","GL","GR","HR","HU","IE","IS","IT","JE","LT","LU","LV","MC","NL","NO","PL","PT","SE","SI","SK","VA","") 
| join countryCSV on $left.countryString == $right.countryCode 
| project TimeGenerated, Identity, UserPrincipalName, countryString, CountryName, IPAddress, cityString, stateString, deviceName, browser 
| distinct Identity, UserPrincipalName, countryString, CountryName, cityString, stateString, TimeGenerated, deviceName, browser 
```

You may note that in this case, you should also exclude "" from the query because it will create a lot of noise that you cannot use.

### Conclusion

You can integrate those queries in any workbook as long as you provide the country list and the proper logs, this is very useful to monitor compliance to travel policies, or monitor suspicious activities from unusual countries.

You can also use those queries to trigger alerts in sentinel, and all this kind of fun :)

