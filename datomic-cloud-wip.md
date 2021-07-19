https://clojurians.slack.com/archives/C03S1KBA2/p1598653040371500

I am using Datomic Cloud and am trying to compile my code before deployment -- to catch any errors without incurring AWS costs. I am using @seancorfield’s excellent depstar but I don't have a -main since I am deploying Ions. Can anyone advise me how to do the compilation? thanks!
7 replies
seancorfield  5 days ago
@hadils Do Ions not have a standard entry point?
seancorfield  5 days ago
(I am not familiar with them)
hadils  5 days ago
No, the entry point is in Datomic Cloud...
seancorfield  5 days ago
Yeah, but how does it know what to call in your Ion code?
seancorfield  5 days ago
The general answer here is:
1. create a classes folder
2. add it to your :paths in deps.edn
3. run clojure -e "(compile,'your.namespace)" to compile stuff
4. run depstar so it will JAR up the source and the classes
seancorfield  5 days ago
Or if you only care about putting the source in the JAR, but want to do the compile as a sanity check, add a :compile alias that has :main-opts ["-e" "(compile,'your.namespace)"] and then clojure -A:compile and don't both putting "classes" in your :paths -- you still need the folder for compile to compile into.
hadils  5 days ago
Thanks @seancorfield! I appreciate your help!
