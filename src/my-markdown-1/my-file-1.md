# My test markdown code

```mermaid
graph LR
  Producers[Data Producers] --> Ingestion
  Ingestion --> Storage[Long-term Storage]
  Ingestion --> Stream[Stream Processing]
  Stream --> Storage
  Batch[Batch Processing] --> Storage
  Storage --> Batch
  Self[Self Serve] -.- Stream
  Self -.- Batch
  Stream -.-> Visualization
  Batch -.-> Visualization
  Stream --> Export
  Batch --> Export
```

Testing MathJax equation format 1

$$
\eta = \sum r
$$

Testing MathJax equation format 2

\\[
\zeta = \sum f
\\]

Testing MathJax inline equation $ \beta \in [a_1..a_n] $ format 1

Testing MathJax inline equation \\(  \gamma \in [b_1..b_n] \\) format 2

```mermaid
%% Example of sequence diagram
sequenceDiagram
    Alice->>Bob: Hello Bob, how are you?
    alt is sick
    Bob->>Alice: Not so good :(
    else is well
    Bob->>Alice: Feeling fresh like a daisy
    end
    opt Extra response
    Bob->>Alice: Thanks for asking
    end
```

text inbetween

```mermaid
graph TD;
    A-->B;
    A-->C;
    B-->D;
    C-->D;
```

