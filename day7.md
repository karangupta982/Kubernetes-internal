# Cert-Manager, Gateway API, Namespaces, TLS

- how communication happens in internet: Two entities client and server, client has session key and server has public key and private key. user needs to connect and communicate, user sends his query then server return it's public key in response then user sends this public key to Certificate authority then certificate authority checks whether the api key is valid for the certificate and if it is then user wraps it's session key with public key of server and sends it to the server and if it is valid then with it's private key it can decrypt the session key and now both user and server have session key, so in this way communication happens.
- Instructor also said something like http works on layer 7 and TLs works on layer 4, so how it happens something like that please explain the hidden concept in this also.
- TLS:?, MTLS:?

- let's encrypt: certificate authority, you will make an api call and that server will tell you whether this certificate is valid or not.
- cert manager: ?
- cluster issuer:?
- certificate and certificate request

- application is running standalone with ip address of service but using ingress application is not running. why is it so?

- Ingress advantages and disadvantage
  - ingress controller can handle only layer 7 load balancer and not layer 4 , layer 5
- Gateway API: why? what? where? advantages? disadvantages? which protocols and methods it works on?
  - gateway class:?
  - Gateway:?
  - HttpRoute:?
 
    Loadbalancer: may be used for not only balancing traffic but also for a lot of work collecting metric, implementing ddos protecting, taking count of unique inputs, latency, failure rate,...
    
  
