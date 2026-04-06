# Context Map

```mermaid
graph LR
    ID["Identity & Session"]
    RG["Room Gameplay"]
    TO["Tournament Orchestration"]
    RS["Ranking & Statistics"]
    SV["Spectator View"]
    AL["Audit & Game Log"]

    ID -->|"U/D, OHS"| RG
    ID -->|"U/D, OHS"| TO
    ID -->|"U/D, OHS"| RS
    ID -->|"U/D, Conformist"| AL

    RG -->|"Partnership"| TO
    RG -->|"U/D, Conformist"| RS
    RG -->|"U/D, ACL - privacy filter"| SV
    RG -->|"U/D, Conformist"| AL

    TO -->|"Partnership"| RG
    TO -->|"U/D, Conformist"| SV
    TO -->|"U/D, Conformist"| AL

    RS -->|"U/D, Conformist"| AL

    SV -->|"U/D, Conformist"| AL
```
