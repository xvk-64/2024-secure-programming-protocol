
This is an example of a simple neighboorhood, you are able to communicate with everyone in the graph, in becuase you are all directly connected.

```mermaid
graph LR;
subgraph connections
A[ClientA] -.-> B((Server1));
C[ClientB] -.-> B;
B <--> D((Server2));
E[ClientC] -.-> D;
end
```
This is an example of a more complex neighboorhood. Again all clients can talk to other clients beucase they either share the same homeServer or are connected to servers directly connected to your homeServer.

```mermaid
graph LR;
subgraph connections
A[ClientA] -.-> B((Server1));
C[ClientB] -.-> B;
B <--> D((Server2));
E[ClientC] -.-> D;
D <--> F((Server3));
G[ClientD] -.-> F;
H[ClientE] -.-> F;
D <--> I((Server4));
J[ClientF] -.-> I;
K[ClientG] -.-> I;
B <--> F;
B <--> I;
F <--> I;
end
```
The following three diagrams explain who you would be able to talk to in a situation, where servers connected and didn't properly form a neighbourhood.

If you are ClientA you can message any Client within the yellow rectangle. ClientA can message ClientB becuase they are on the same server and can message ClientC, becuase their home servers have a direct connection. 
```mermaid
graph LR;
subgraph connections
A[ClientA] -.-> B((Server1));
C[ClientB] -.-> B;
B <--> D((Server2));
E[ClientC] -.-> D;
end
D <--> F((Server3));
G[ClientD] -.-> F;
H[ClientE] -.-> F;
D <--> I((Server4));
J[ClientF] -.-> I;
K[ClientG] -.-> I;
```
If you are ClientC, you can message anyone within the yellow rectangle. This is becuase the clientC's homeServer, Server2, is directly connected to all the other servers in this neighbourhood. 
```mermaid
graph LR;
subgraph connections
A[ClientA] -.-> B((Server1));
C[ClientB] -.-> B;
B <--> D((Server2));
E[ClientC] -.-> D;
D <--> F((Server3));
G[ClientD] -.-> F;
H[ClientE] -.-> F;
D <--> I((Server4));
J[ClientF] -.-> I;
K[ClientG] -.-> I;
end
```
If you are ClientE you can message any client within the yellow rectangle. ClientE can message ClientD becuase they are on same the server and can message ClientC, becuase their home servers have a direct connection.

```mermaid
graph LR;

A[ClientA] -.-> B((Server1));
C[ClientB] -.-> B;
B <--> D((Server2));
subgraph connections
E[ClientC] -.-> D;
D <--> F((Server3));
G[ClientD] -.-> F;
H[ClientE] -.-> F;
end
D <--> I((Server4));
J[ClientF] -.-> I;
K[ClientG] -.-> I;

```

