# YAML
- Yaml is case sensitive.
- Data -> Byte stream + conversion content to gain the exact format again -> serialization
- Yaml has :, -, space, english
- In place of yaml, json, protobuf and xml can be used.
- Yaml is used only because it is human readable configuration and data storage.
- yaml is superset of json.
- camelcase: karanGupta (Generally in kubernetes camelCasing is used).
- YAML = YAML Ain't Markup language. It is not a language but a declarative way to define things. It is not needed to tell yaml that store it in int, float, double. It is capable enough to detect that.
- name:"karan" , double quotes are not needed in yaml until you are using special characters like n, &.
- Yaml is a key-value pair, Key is unique but value may be infinite. For example name:Karan, now name can't be used one more time, Name:Karan can be used since yaml is case sensitive.
- we can store yaml as .yaml and .yml both ways.
- --- is used when we have multiple yamls in a single file.
- hyphen means it is a list, still i didn't understand what exactly is mean by hyphen.
- Comments are added using #.


  # LINUX for kubernetes point of view
  - ls -a : shows hidden files
  - ps aux: list down running process
  - ps tree: gives hierarchical view of folder
  - base64 format is used in kubernetes secrets, conversion to base64, echo karan | base64  -> it will return base64 encoding of karan and conversion from base64 to string:
   for example: echo karan | base64 response is cHJ1cm10Cg==
   then echo cHJ1cm10Cg== | base64 --decode  will decode back.
 
  - head to see top 10 lines in any configuration file
  - tail is used to see last 10 lines in any log file
  - ctrl + c is exit in linux
  - tcpdump commands
  - crictl : to directly talk with pods
  - crictl pods, crictl ps -a, sudo crictl logs id
  - systemd linux ki sari commands chalata h, but what is systemd
  - system etc ke ander store hota h
  - sudo journalctl --no-pager, sudo journalctl --since yesterday
  - -o means output
  - sudo journalctl --since yesterday -o json-pretty
 

  # Kubernetes
  - kubernetes releases two times a year (one release cycle is of four months)
  - Always remember the current version of kubernetes, important for inerview.
  - cluster -> Node -> Pod -> Container






