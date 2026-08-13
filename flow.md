Client request → hits the Load Balancer DNS.

Listener → receives the request on port 80.

Listener forwards to Target Group → the LB only knows “send this to my target group.”

Target Group → has a list of registered EC2s (your attachments).

Load Balancer algorithm → looks at the EC2s in the target group, checks their health, and decides which one to use.

Default is round‑robin (alternate between instances).

If one EC2 fails health checks, it’s skipped.

Chosen EC2 → serves the response back through the LB to the client.