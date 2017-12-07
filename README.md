# Background

The provisioner is an application to generate provisioning information for hardware VoIP phones. It is written in php and uses [twig](https://twig.sensiolabs.org/) templates. It has been tested with the following phones: 
* Cisco SPA3x and SPA5x
* Yealink T2x and T4x
* Polycom SoundPoint and VVX
* Grandstream GXP2170 and GXP2130

# Data flow

When a device is created or updated through crossbar, the device data are slightly reformatted and copied to the provisioner database. In addition the account document is also copied to the provisioner. Refer to [the diagram](https://www.gliffy.com/go/publish/12121940).

# Installation for CentOS 7 with httpd

Prepare you fresh CentOS 7 server.
```
yum -y install httpd mod_ssl git epel-release
```

## Clone the repo
```
cd /var/www/html
git clone https://github.com/OpenTelecom/provisioner.git
```

## Configure httpd
### Create the httpd conf file
```
vim /etc/httpd/conf/kazooprovision.conf
<VirtualHost *:80>
        ServerName provisioning-server-domain
        ServerAlias provisioning-provider01-domain provisioning-provider02-domain
        ServerAdmin webmaster@yourdomain.foundation
        DocumentRoot /var/www/html/provisioner/
        Timeout 600
        DirectoryIndex index.php index.html
        <Directory />
                Options FollowSymLinks
                AllowOverride All
        </Directory>
</VirtualHost>
```
Change ```provisioning-server-domaim```, ```provisioning-provider01-domain```, ```provisioning-provider02-domain``` ```webmaster@yourdomain.foundation``` ```/var/www/html/provisioner/``` as appropriate.

### Load the configuration

```systemctl reload httpd```

## Configure the provisioner
### Create and update the config.json file
```
cp /var/www/html/provisioner/config_sample.json /var/www/html/provisioner/config.json
```

Update ```config.json``` with the appropriate settings:

Set the value for ```"adapter"``` to "2600hz".

Set the value for ```"db_prefix"```. Choose a value that all provisioner Couch databases will be prefixed with e.g. zz_provisioner.

Replace ```my.domain.com``` with the domain name of the provisioning server.

Replace ```my.bigcouch-server.com``` with the domain name of the Couch server where the provisioner databases will be stored.

Replace ```Master provider``` with the name of the provider. This is an arbitrary value and can be set to anything.

Replace ```MyIP``` with the IP address of the provisioning server.

Replace ```MyDomain``` with the domain name of the provisioning server.

### Create the necessary provisioner Couch databases 

```php setup_db.php```

This will create the following databases (with the prefix as set in the ```config.json``` file):
* ```db_prefix```factory_defaults: Contains default settings at make, family and model level.
* ```db_prefix```mac_lookup: Contains document for each MAC address which maps to the account id.
* ```db_prefix```providers: Contains a document for each provider. This allows a set of authorised IPs and configuration settings to be set per domain name.
* ```db_prefix```system_account: Contains default settings at system level.

### Create provisioner providers 
Create a document in the provisioner providers Couch database for each provider. You may create one or more providers.
```
{
   "_id": "PROVIDED-BY-COUCHDB",
   "name": "CloudPBX",
   "authorized_ip": [
       "::0",
       "127.0.0.1",
       "crossbar-public-ip",
       "crossbar-public-ip",
       "crossbar-public-ip",
       "crossbar-public-ip",
       "crossbar-public-ip",
       "crossbar-public-ip"
   ],
   "domain": "provisioning-provider01-domain",
   "default_account_id": null,
   "pvt_access_type": "admin",
   "pvt_type": "provider",
   "settings": {
       "outbound_proxy": {
           "enable": "1",
           "primary": {
               "host": "kamailio.domain"
           }
       },
       "wallpapersource": "1",
       "wallpaperserverpath": "www.yourdomain.com/reseller.jpg",
       "hideblfremotestatus": "1"
       "acceptincomsipfromproxyonly1": "1",
       "acceptincomsipfromproxyonly2": "1",
       "acceptincomsipfromproxyonly3": "1",
       "acceptincomsipfromproxyonly4": "1",
       "acceptincomsipfromproxyonly5": "1",
       "acceptincomsipfromproxyonly6": "1",
       "userandomport": "1",
   }
}
```
Replace ```provisioning-provider01-domain``` and ```provisioning-provider02-domain``` with the domain names of your providers. 

Replace ```Provider Name``` with the name of this provider. This is an arbitrary value and can be set to anything.

Replace ```crossbar-public-ip``` with the IP of the crossbar server that will be communicating with the provisioner.

Replace ```kamailio.domain``` with the domain name or IP of the Kamailio server that devices will authenticate with.

Replace ```wallpapersource value``` with Wallpaper Source. O - Default, 1 - Download, 2 - USB, 3 - Uploaded (for grandstream)

Replace ```wallpaperserverpath value``` with Wallpaper Server Path (for grandstream)

Replace ```hideblfremotestatus value``` with Hide BLF Remote Status. 0 - No, 1 - Yes. Default is 0-11 (for grandstream)

Replace ```acceptincomsipfromproxyonly1 value``` with Account 1 Accept Incoming SIP from Proxy Only value (for grandstream)

Replace ```acceptincomsipfromproxyonly2 value``` with Account 2 Accept Incoming SIP from Proxy Only value (for grandstream)

Replace ```acceptincomsipfromproxyonly3 value``` with Account 3 Accept Incoming SIP from Proxy Only value (for grandstream)

Replace ```acceptincomsipfromproxyonly4 value``` with Account 4 Accept Incoming SIP from Proxy Only value (for grandstream)

Replace ```acceptincomsipfromproxyonly5 value``` with Account 5 Accept Incoming SIP from Proxy Only value (for grandstream)

Replace ```acceptincomsipfromproxyonly6 value``` with Account 6 Accept Incoming SIP from Proxy Only value (for grandstream)

Replace ```userandomport value``` with Use Random Port value (for grandstream)

Provisioner providers settings value can be overwrite at provisionaccount/$accountid/$MAC document. eg: put '"wallpaperserverpath": "www.yourdomain.com/reseller2.jpg",' at provisionaccount/$accountid/$MAC document setting field, it will overwrite the value at provisioner providers document.

### Create phone make, family and model details

```
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco/spa5xx
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco/spa5xx/504g
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco/spa5xx/303
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco/spa5xx/501g
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco/spa5xx/502g
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco/spa5xx/525g
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/cisco/spa5xx/525g2

curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/polycom
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/polycom/vvx
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/polycom/vvx/201
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/polycom/vvx/300

curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t2x
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t2x/t20
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t2x/t22
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t2x/t26
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t2x/t28
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t2x/t19
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t4x
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t4x/t46
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/yealink/t4x/t42

curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/grandstream/
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/grandstream/gxphd
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/grandstream/gxphd/gxp2170
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/grandstream/gxphd/gxp2130
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/grandstream/gac
curl -i -H "Accept: application/json" -H "Content-Type: application/json" -X PUT -d '{"settings": {}}' http://provisionerurl/api/phones/grandstream/gac/gac2500
```


### Test the provisioner
Visit http://provisioning-server-domain/api/phones and a list of phones should be returned
e.g.
```
{
  "data": {
    "cisco": {
      "id": "cisco",
      "name": "cisco",
      "settings": [
        
      ],
      "families": {
        "spa": {
          "id": "cisco_spa",
          "name": "spa",
          "settings": [
            
          ],
          "models": {
            "901": {
              "id": "cisco_spa_901",
              "name": "901",
              "settings": [
                
              ]
            },
            ...
```
## Configure Monster UI

### Update config.js
```define(function(require) {

        return {
                api: {
                        provisioner: 'http://provisioning-server-domain/api/',
...
```

Replace ```provisioning-server-domain``` with the domain name of your provisioner.

### Test Monster UI

Navigate to Smart PBX > Devices > Add Device > SIP Phone. You should be prompted to select the device brand.

## Configure Kazoo

Configure the appropriate settings in the ```crossbar.devices``` document available in your Kazoo HAProxy server at http://127.0.0.1:15984/_utils/document.html?system_config/crossbar.devices
```{
   "_id": "crossbar.devices",
   "default": {
       "provisioning_type": "super_awesome_provisioner",
       "provisioning_url": "http://provisioning-server-domain/api/accounts",
       "allow_aggregates": "true"
   }
}
```

Replace ```provisioning-server-domain``` with the domain name of the provisioner.

## Specify IP ACLs
Provisioner checks whether the ```access_lists``` key exists on the account document. If one or more CIDRs exist then the provisioner validates the IP and will allow or deny the provisioning request as appropriate.

For example given the following access list for an account:
```
   "access_lists": {
       "order": "allow,deny",
       "cidrs": [
           "8.8.8.8/32",
           "8.8.4.4/32"
       ]
       }
```

Any requests for provisioning files from 8.8.8.8 or 8.8.4.4 for a MAC ID that is assigned to that account will be allowed. Requests from any other IP will be denied.

## Create Kazoo device

### Background
The following keys should be populated in the device document in order for the provisioning data to be generated:

```sip``` contains the username and password for the first account on the phone

```mac_address``` the MAC address for the phone

```provision.endpoint_brand``` the phone brand e.g. yealink, cisco, polycom

```provision.endpoint_family``` the phone family e.g. t2x, spa5xx

```provision.endpoint_model``` the phone model e.g. t26, spa303

```provision.settings.accounts``` contains the username, password, domain and proxy details for the second and subsequent accounts on the phone

```provision.settings.lines``` contains the mapping from the line keys on the phone to various functions. The function is determined by the type setting.

```provision.settings.combo_keys``` contains the mapping from the combo keys on the phone to various functions. The function is determined by the type setting.

```provision.settings.sidecar``` contains configuration for settings that are common across all sidecars. The function is determined by the type setting.

```provision.settings.sidecar_01``` contains the mapping from the keys on the first sidecar to various functions. The function is determined by the type setting.

```type``` One of the following values:

Yealink
* 13: speed dial
* 15: account
* 16: BLF

Grandstream
*Please note that each model has a set number of Fixed VPK and Dynamic VPK. Based on fixed/dynamic, your type will change.
* 0: account
* 11: BLF (Fixed only)
* 1: BLF (Dynamic only)

### Example Yealink device document
```
{
  "data": {
    "sip": {
      "password": "passw0rd",
      "username": "user_abcd",
      "expire_seconds": 300,
      "invite_format": "username",
      "method": "password"
    },
    "device_type": "sip_device",
    "enabled": true,
    "mac_address": "00:15:15:15:15:15",
    "name": "test t26",
    "owner_id": "cd7ca46d83a38b7f02a8e1b73f8a463f",
    "provision": {
      "endpoint_brand": "yealink",
      "endpoint_family": "t2x",
      "endpoint_model": "t26",
      "settings": {
        "accounts": {
          "2": {
            "basic": {
              "enable": true,
              "display_name": "test 2 t26"
            },
            "sip": {
              "username": "user_4abcj",
              "password": "1234",
              "realm_01": "1000009.yourdomain.foundation",
              "outbound_proxy_01": "sip.yourdomain.foundation",
              "transport": "1"
            }
          }
        },
        "lines": {
          "1": {
            "type": "15",
            "key": {
              "line": "1",
              "value": "1593",
              "label": "1593"
            }
          },
          "2": {
            "type": "15",
            "key": {
              "line": "2",
              "value": "1594",
              "label": "1594"
            }
          },
          "3": {
            "type": "16",
            "key": {
              "line": "2",
              "value": "2009",
              "label": "2009"
            }
          }
        },
        "combo_keys": {
          "1": {
            "type": "16",
            "key": {
              "line": "1",
              "value": "1596",
              "label": "1596"
            }
          },
          "2": {
            "type": "16",
            "key": {
              "line": "1",
              "value": "1599",
              "label": "1599"
            }
          },
          "3": {
            "type": "13",
            "key": {
              "line": "1",
              "value": "5551231234",
              "label": "5551231234"
            }
          }
        },
        "sidecar_01": {
          "1": {
            "type": "16",
            "key": {
              "line": "1",
              "value": "1593",
              "label": "1593"
            }
          },
          "2": {
            "type": "16",
            "key": {
              "line": "2",
              "value": "1594",
              "label": "1594"
            }
          },
          "3": {
            "type": "16",
            "key": {
              "line": "2",
              "value": "1595",
              "label": "1595"
            }
          }
        }
      }
    }
   }
}
```
### Example Yealink device explained

* Two accounts are configured one with username user_abcd and the other with username user_4abcj
* Three lines are configured. Line 1 is linked to account 1, line 2 is linked to account 2, line 3 is set to monitor BLF on extension 2009 on account 2
* Three combo_keys are configured. Key 1 is set to monitor BLF on extension 1596 on account 1, key 2 is set to monitor BLF on extension 1599 on account 1, key 3 is set to speed dial 5551231234 using account 1.
* Three buttons on sidecar are configured. Key 1 is set to monitor BLF on extension 1593 on account 2, key 2 is set to monitor BLF on extension 1594 on account 2, key 3 is set to monitor BLF on extension 1595 on account 2.

### Example Grandstream device document
```
{
  "data": {
   "brand": "grandstream",
   "family": "gxphd",
   "model": "gxp2170",
    "sip": {
      "password": "passw0rd",
      "username": "user_abcd",
      "expire_seconds": 300,
      "invite_format": "username",
      "method": "password"
    },
    "device_type": "sip_device",
    "enabled": true,
    "mac_address": "00:15:15:15:15:15",
    "name": "test gxp2160",
    "owner_id": "cd7ca46d83a38b7f02a8e1b73f8a463f",
    "provision": {
      "endpoint_brand": "grandstream",
      "endpoint_family": "gxphd",
      "endpoint_model": "gxp2170",
      "settings": {
       "wallpapersource": "1",
       "wallpaperserverpath": "www.yourdomain.com/reseller.jpg",
       "accounts": {
          "1": {
            "basic": {
              "enable": true,
              "display_name": "test 2 gxp2170"
            },
            "sip": {
              "username": "user_4abcj",
              "password": "1234",
              "realm_01": "1000009.yourdomain.foundation",
              "outbound_proxy_01": "sip.yourdomain.foundation"
            }
          }
        },
        "lines": {
          "1": {
            "type": "0",
            "key": {
              "line": "0",
              "value": "1593",
              "label": "1593"
            }
          },
          "2": {
            "type": "0",
            "key": {
              "line": "1",
              "value": "1594",
              "label": "1594"
            }
          },
          "3": {
            "type": "11",
            "key": {
              "line": "1",
              "value": "2009",
              "label": "2009"
            }
          },
          "mpk1": {
            "type": "16",
            "key": {
              "line": "0",
              "value": "*3101",
              "label": "Park 101"
            }
          }
        },
        "sidecar_01": {
          "1": {
            "type": "11",
            "key": {
              "line": "0",
              "value": "1593",
              "label": "1593"
            }
          },
          "2": {
            "type": "11",
            "key": {
              "line": "1",
              "value": "1594",
              "label": "1594"
            }
          },
          "3": {
            "type": "11",
            "key": {
              "line": "1",
              "value": "1595",
              "label": "1595"
            }
          }
        }
      }
    }
   }
}
```

### Example Grandstream device explained

* Two accounts are configured one with username user_abcd and the other with username user_4abcj
* Three lines are configured. Line 1 is linked to account 0, line 2 is linked to account 1, line 3 is set to monitor BLF on extension 2009 on account 
* For GXP2130/GXP2160, MPK key 1 is set to monitor parking lot.
* Three buttons on sidecar are configured. Key 1 is set to monitor BLF on extension 1593 on account 0, key 2 is set to monitor BLF on extension 1594 on account 1, key 3 is set to monitor BLF on extension 1595 on account 1.

# Known issues
