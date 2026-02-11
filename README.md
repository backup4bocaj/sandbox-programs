# sandbox-programs
elite-vmi-lab/
│
├── host-setup/
│   ├── install_kvm.sh
│   ├── install_libvmi.sh
│
├── vm-control/
│   ├── snapshot_restore.sh
│   ├── vm_cycle.sh
│
├── introspection/
│   ├── CMakeLists.txt
│   ├── process_list.c
│   ├── syscall_monitor.c (safe logging only)
│
├── telemetry/
│   ├── logger.py
│
└── README.md
#!/bin/bash

git clone https://github.com/libvmi/libvmi.git
cd libvmi
mkdir build
cd build
cmake ..
make -j$(nproc)
sudo make install
sudo ldconfig

echo "LibVMI installed."
#!/bin/bash

VM_NAME="sandbox-vm"
SNAPSHOT_NAME="baseline"

virsh snapshot-revert $VM_NAME $SNAPSHOT_NAME --running

echo "Snapshot restored."
#!/bin/bash

VM_NAME="sandbox-vm"

echo "Starting VM..."
virsh start $VM_NAME

sleep 60

echo "Stopping VM..."
virsh shutdown $VM_NAME
cmake_minimum_required(VERSION 3.5)
project(vmi_process_list)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBVMI REQUIRED libvmi)

include_directories(${LIBVMI_INCLUDE_DIRS})
link_directories(${LIBVMI_LIBRARY_DIRS})

add_executable(process_list process_list.c)
target_link_libraries(process_list ${LIBVMI_LIBRARIES})
#include <stdio.h>
#include <libvmi/libvmi.h>

int main(int argc, char **argv) {
    vmi_instance_t vmi = NULL;

    if (argc != 2) {
        printf("Usage: %s <vm_name>\n", argv[0]);
        return 1;
    }

    if (VMI_FAILURE == vmi_init_complete(&vmi, argv[1],
        VMI_INIT_DOMAINNAME | VMI_INIT_EVENTS,
        NULL,
        VMI_CONFIG_GLOBAL_FILE_ENTRY,
        NULL,
        NULL)) {

        printf("Failed to init LibVMI\n");
        return 1;
    }

    addr_t list_head;
    if (VMI_FAILURE == vmi_read_addr_ksym(vmi,
        "PsActiveProcessHead",
        &list_head)) {

        printf("Failed to locate process list\n");
        vmi_destroy(vmi);
        return 1;
    }

    printf("Process list head: 0x%lx\n", list_head);

    vmi_destroy(vmi);
    return 0;
}
import json
import datetime

def log_event(event):
    record = {
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "event": event
    }

    with open("telemetry_log.json", "a") as f:
        f.write(json.dumps(record) + "\n")

if __name__ == "__main__":
    log_event("VM introspection cycle complete")
# Elite VMI Lab (Educational Research Use Only)

This project demonstrates:

- KVM-based VM isolation
- Snapshot restoration
- LibVMI process enumeration
- External telemetry logging

## Requirements
- Ubuntu 22.04+
- KVM enabled CPU
- libvirt
- LibVMI

## Usage

1. Install host dependencies
2. Create sandbox VM
3. Take baseline snapshot
4. Run introspection tools
5. Restore snapshot after testing
                 INTERNET
                     │
              ┌──────▼──────┐
              │ Gateway Node │
              │ (Ingress)    │
              └──────┬──────┘
                     │
          All files / traffic mirrored
                     │
              ┌──────▼──────┐
              │ Detonation   │
              │ Sandbox VM   │
              └──────┬──────┘
                     │
         Behavior Analysis Engine
                     │
         ┌───────────┴───────────┐
         │                       │
      Allowed                 Blocked
Baseline Snapshot
↓
Inject Sample
↓
Run for 120 seconds
↓
Collect telemetry
↓
Export logs
↓
Revert snapshot
↓
Ready for next sample
                ┌─────────────────────────┐
                │  Physical Host Machine  │
                │  (Hardened + Minimal)   │
                └──────────┬──────────────┘
                           │
                 Hypervisor (KVM/Proxmox)
                           │
      ┌────────────────────┼────────────────────┐
      │                    │                    │
 Sandbox VM 1        Sandbox VM 2        Simulation VM
 (File detonation)    (Behavior lab)     (Model execution)
      │                    │                    │
      └────────────── Telemetry Engine ─────────┘
                           │
                    Logging + Scoring
                           │
                     Snapshot Manager
Entity attempts entry →
Boundary detection →
Redirect to sandbox VM →
Behavior execution →
Score →
Allow or reset
./sandbox-run simulation.py
Restore baseline snapshot
↓
Inject executable or simulation
↓
Start execution
↓
Monitor CPU / memory / syscalls
↓
If threshold exceeded → kill VM
↓
Export telemetry
↓
Revert snapshot
Host (minimal)
    │
    └── Simulation VM Template (baseline snapshot)
            │
            ├── Execution wrapper
            ├── Resource limiter
            ├── Telemetry agent
            └── Auto-reset
#!/bin/bash

VM="sim-vm"
SNAP="baseline"
SIM_FILE=$1

if [ -z "$SIM_FILE" ]; then
    echo "Usage: ./sim-run.sh simulation.py"
    exit 1
fi

echo "[+] Restoring snapshot"
virsh snapshot-revert $VM $SNAP --running

sleep 5

echo "[+] Copying simulation into VM"
virt-copy-in -d $VM $SIM_FILE /home/user/

echo "[+] Executing simulation"
virsh qemu-agent-command $VM \
'{"execute":"guest-exec","arguments":{"path":"/usr/bin/python3","arg":["/home/user/'"$SIM_FILE"'"]}}'

sleep 60

echo "[+] Shutting down VM"
virsh shutdown $VM
<memory unit='MiB'>2048</memory>
<vcpu>2</vcpu>
<cputune>
    <quota>50000</quota>
    <period>100000</period>
</cputune>
#!/bin/bash

VM="sandbox-vm"
SNAP="baseline"
TARGET=$1

if [ -z "$TARGET" ]; then
    echo "Usage: ./sandbox-run.sh program"
    exit 1
fi

virsh snapshot-revert $VM $SNAP --running
sleep 5

virt-copy-in -d $VM $TARGET /home/user/

virsh qemu-agent-command $VM \
'{"execute":"guest-exec","arguments":{"path":"/home/user/'"$TARGET"'"}}'

sleep 60

virsh shutdown $VM
chmod -x /usr/bin/python3
chmod -x /usr/bin/gcc
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 1
net.ipv4.conf.all.rp_filter = 1
class DigitalAirspace:

    def __init__(self):
        self.threshold = 70

    def boundary_cross(self, entity):
        print(f"[+] Entity {entity['name']} entering airspace")
        score = self.evaluate(entity)

        if score > self.threshold:
            print("[-] Entity contained in sandbox.")
            return "contained"
        else:
            print("[+] Entity allowed.")
            return "allowed"

    def evaluate(self, entity):
        risk = entity.get("cpu_usage", 0) + entity.get("network_calls", 0)
        return risk


airspace = DigitalAirspace()

sample = {
    "name": "simulation_X",
    "cpu_usage": 50,
    "network_calls": 30
}

airspace.boundary_cross(sample)

