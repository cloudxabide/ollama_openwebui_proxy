# README

I am documenting my approach to adding a proxy to frontend my AI resources hosted locally in my DMZ.

![Cartoon](Images/high-level-cartoon-by_Gemini.png)

* provide proxy for different hosts (hostname-based redirect)
* add authentication for Ollama
* terminate SSL (seemed like I might as well)

I have a border gateway/firewall which I DNAT the standard ports to my HAproxy node (11434/12000 - Ollama/OpenWebUI).  From there I direct the ingress traffic to the appropriate node based on hostname (jarivs - PC with RTX card, spark-E - NVIDIA DGX Spark).

![Network Overview](Images/high-level-ollama_openwebui.png)






