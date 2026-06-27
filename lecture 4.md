Youtube video Link: https://www.youtube.com/watch?v=A7F7lKzfcsM&t=573s


Image of an application is build on base images, there are multiple layers in image and sometimes in some layer there might be CVE (Common Vulnerability and exposure), each layer has changes for having CVE, we want that each layer should be secure so that no hacker could enter in container (How hacker enters and does everything), So for solving this we use trivy tool, it has all the CVE stored. Trivy can be integrated standalone or in CICD. Our aim is to minimize CVE's. Three types of CVE, Critical, High, Low.
commands: trivy image nginx --severity CRITICAL
For security issues we use:
  - Multi stage build
  - Distroless images
  - Trivy tool

# Bat
Beutiful coloring and good representation of file instead of using cat command.

 # What happens when kubectl command is hit
 ## i write kubectl command -> it goes to api server -> api server does authentication, authorization then admission controllers check with mutating controllers then validating controllers then ...(Don't know what happens after this, explain it clear first then take an example and then dry run the explaination)

# Kyverno
### What is it? why does it exist? what it is used for? What it actually does when run in real time like when instructor run it, it was creating: CRDs (what is it?), Admission Controller, Report controller, Cleanup controller, Background controller.



