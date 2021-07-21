makecert.exe -n "CN=XXX" -r -pe -a sha512 -len 4096 -cy authority -sv CARoot.pvk CARoot.cer

pvk2pfx.exe -pvk CARoot.pvk -spc CARoot.cer -pfx CARoot.pfx -po password

makecert.exe -n "CN=XXX" -iv CARoot.pvk -ic CARoot.cer -pe -a sha512 -len 4096 -b 01/01/2021 -e 01/01/2060 -sky exchange -eku 1.3.6.1.5.5.7.3.2 -sv client.pvk client.cer


pvk2pfx.exe -pvk Client.pvk -spc Client.cer -pfx Client.pfx -po password

