# Advanced Docker Course Site

## lab VMs
<table>
<tr><th>vm number</th><th>public ip</th></tr>
<tr><td>lab1</td> <td>35.178.180.210</td></tr>
<tr><td>lab2</td> <td>18.130.250.20</td></tr>
<tr><td>lab3</td> <td>18.130.231.4</td></tr>
<tr><td>lab4</td> <td>52.56.136.61</td></tr>
<tr><td>lab5</td> <td>35.178.205.155</td></tr>
<tr><td>lab6</td> <td>52.56.218.173</td></tr>
<tr><td>instructor</td> <td>52.56.158.75</td></tr>
</table>

### SSH 
The SSH private key is available in `keys` directory. 

To SSH into the VM please make sure permissions are set correctly on the key

```
chmod 600 /path/to/lab
```

The username for SSH is `ubuntu`

```
ssh -i /path/to/lab ubuntu@<labvmIP>

```

## Course work

## Day 1
Lab 1: [compose](labs/1-compose/)  
Lab 2: [union-filesystem](labs/2-union-filesystem/)  
Lab 3: [cgroups-namespaces](labs/3-cgroups-namespaces/)  
Lab 4: [builds](labs/4-builds/)  

## Day 2
Lab 5: [security](labs/5-security/)  
Lab 6: [networks](labs/6-networks/)  
Lab 7: [api](labs/7-api/)  
Lab 8: [orchestration](labs/8-orchestration/)  


## Course Content
[Day 1](https://www.dropbox.com/s/j6ejnnofymyo5sd/adv-docker-day1.pdf?dl=0)  
[Day 2](https://www.dropbox.com/s/7jwomtui7rdisvo/adv-docker-day2.pdf?dl=0)


## Survey
[Please provide feedback](http://www.metricsthatmatter.com/student/evaluation.asp?k=16324&i=ILT00416277)
