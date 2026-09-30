```mermaid
flowchart TD
    A[Text Editor] --> B[Code & Data from GitHub]
    B --> C[PostgreSQL Database]
    C --> D[pgAdmin GUI]
``` 

---

## Why  Text Editor?

```mermaid
flowchart TD
    A[CSV File] --> B{Open With?}
    B -->|Excel / Word| C[❌ Data gets altered]
    B -->|Text Editor| D[✅ Data stay exact]
```

---

PostgreSQL  **database server**. pgAdmin **graphical control panel**.

```mermaid
flowchart LR
    subgraph Linux
        L1[apt/yum packages] --> L2[PostgreSQL + pgAdmin + PostGIS + PL/Python]
    end
```

---

> "Proper planning prevents poor performance."

Setting up correctly **avoids headache later**.