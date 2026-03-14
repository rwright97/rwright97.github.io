# T-Pot Honey Pot Deployment Lab<br>
#### This lab's purpose for me is to setup security researching so that I can monitor, and study the attacks that happen on a Honeyot.
#### Through studying active attacks that have occured, as a Blue team proffesional you can better pickup on indicators and threat actor artifacts left behind from attacks.

### Network Architecture<br>
flowchart TD
    %% Define the outer internet and firewall
    Cloud((Internet / Threat Actors)) -- "Targets AT&T Static IP\n(1:1 NAT)" --> OPNsense{{OPNsense Firewall}}

    %% Define the secure LAN zone
    subgraph SafeZone [Internal LAN Network - Safe]
        Switch[Unmanaged Switch]
        Admin[SysAdmin PC / Management]
    end

    %% Define the Proxmox Server
    subgraph Proxmox [Proxmox Bare-Metal Server]
        direction TB
        
        subgraph SecureHost [Host Management]
            NIC1[NIC 1] --- VMBR0[vmbr0 Bridge]
        end
        
        subgraph DangerZone [DMZ Air-Gap]
            NIC2[NIC 2] --- VMBR1[vmbr1 'Invisible' Bridge\nNO IP ASSIGNED]
            VMBR1 --- Tpot[T-Pot VM\nIP: 10.99.99.50]
        end
    end

    %% Wiring Connections
    OPNsense -- "LAN Port" --> Switch
    Switch -- "Safe Network Traffic" --> NIC1
    Switch --- Admin

    OPNsense -- "OPT1 Port (DMZ)\nDedicated Physical Cable" --> NIC2

    %% Styling
    classDef danger fill:#f8d7da,stroke:#dc3545,stroke-width:2px,color:#721c24;
    classDef safe fill:#d4edda,stroke:#28a745,stroke-width:2px,color:#155724;
    classDef firewall fill:#cce5ff,stroke:#004085,stroke-width:2px,color:#004085;

    class DangerZone,Tpot,NIC2,VMBR1 danger;
    class SafeZone,Switch,Admin,NIC1,VMBR0,SecureHost safe;
    class OPNsense firewall;
