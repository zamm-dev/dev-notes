# ngrok

Per the [setup instructions](https://dashboard.ngrok.com/get-started/setup/macos) on Mac:

```bash
$ brew install ngrok/ngrok/ngrok
```

Then

```bash
$ ngrok config add-authtoken <TOKEN>
Authtoken saved to configuration file: /Users/amos/Library/Application Support/ngrok/ngrok.yml
```

Then

```bash
$ ngrok http http://localhost:4321
ngrok                                                           (Ctrl+C to quit)
                                                                                
Help shape K8s Bindings https://ngrok.com/new-features-update?ref=k8s           
                                                                                
Session Status                online                                            
Account                       Amos Ng (Plan: Free)                              
Version                       3.18.2                                            
Region                        Asia Pacific (ap)                                 
Latency                       45ms                                              
Web Interface                 http://127.0.0.1:4040                             
Forwarding                    https://b0fa-175-100-59-206.ngrok-free.app -> http
                                                                                
Connections                   ttl     opn     rt1     rt5     p50     p90       
                              0       0       0.00    0.00    0.00    0.00      

```

and you have https for your local server.
