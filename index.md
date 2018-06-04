# Advanced Docker Course Site

## lab VMs
<table>
<tr><th>vm number</th><th>public ip</th></tr>
<tr><td>lab1</td> <td>34.239.183.99</td></tr>
<tr><td>lab2</td> <td>34.227.68.89</td></tr>
<tr><td>lab3</td> <td>34.207.238.200</td></tr>
<tr><td>lab4</td> <td>54.236.13.162</td></tr>
<tr><td>lab5</td> <td>34.230.24.65</td></tr>
<tr><td>lab6</td> <td>54.90.225.67</td></tr>
<tr><td>lab7</td> <td>35.173.179.86</td></tr>
<tr><td>lab8</td> <td>35.153.139.54</td></tr>
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
Lab 2: [cgroups-namespaces](labs/2-cgroups-namespaces/)  
Lab 3: [builds](labs/3-builds/)  

## Day 2
Lab 4: [security](labs/4-security/)  
Lab 5: [networks](labs/5-networks/)  
Lab 6: [api](labs/6-api/)  
Lab 7: [orchestration](labs/7-orchestration/)  


## Course Content
[Day 1](https://www.dropbox.com/s/j6ejnnofymyo5sd/adv-docker-day1.pdf?dl=0)  
[Day 2](https://www.dropbox.com/s/7jwomtui7rdisvo/adv-docker-day2.pdf?dl=0)

## Feedback
[Survey](http://www.metricsthatmatter.com/student/evaluation.asp?k=16324&i=ILT00428249)


