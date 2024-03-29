Libvirt VM
==========

This role configures and creates (or destroys) VMs on a KVM hypervisor.

Requirements
------------

The host should have Virtualization Technology (VT) enabled and should
be preconfigured with libvirt/KVM.

Role Variables
--------------

- `libvirt_vm_default_console_log_dir`: The default directory in which to store
  VM console logs, if a VM-specific log file path is not given. Default is
  "/var/log/libvirt/qemu/".

- `libvirt_vm_image_cache_path`: The directory in which to cache downloaded
  images. Default is "/tmp/".

- `libvirt_volume_default_images_path`: Directory in which instance images are
  stored. Default is '/var/lib/libvirt/images'.

- `libvirt_volume_default_type`: What type of backing volume does the instance use? Default is `volume`.

- `libvirt_volume_default_format`: Format for volumes created by the role, Default is `qcow2`.

- `libvirt_volume_default_device`: Control how device appears in guest OS. Defaults to `disk`.


- `libvirt_vm_engine`: virtualisation engine. If not set, the role will attempt
  to auto-detect the optimal engine to use.

- `libvirt_vm_emulator`: path to emulator binary. If not set, the role will
  attempt to auto-detect the correct emulator to use.

- `libvirt_cpu_mode_default`: The default CPU mode if `libvirt_cpu_mode` or
  `vm.cpu_mode` is undefined.

- `libvirt_vm_arch`: CPU architecture, default is `x86_64`.

- `libvirt_vm_uri`: Override the libvirt connection URI. See the
  [libvirt docs](https://libvirt.org/remote.html) docs for more details.

- `libvirt_vm_virsh_default_env`: Variables contained within this dictionary are
  added to the environment used when executing virsh commands.

- `libvirt_vm_clock_offset`. If defined the instances clock offset is set to
  the provided value. When undefined sync is set to `localtime`.

- `libvirt_vms`: list of VMs to be created/destroyed. Each one may have the
  following attributes:

    - `state`: set to `present` to create or `absent` to destroy the VM.
      Defaults to `present`.

    - `name`: the name to assign to the VM.

    - `memory_mb`: the memory to assign to the VM, in megabytes.

    - `vcpus`: the number of VCPU cores to assign to the VM.

    - `machine`: Virtual machine type. Default is `None` if
      `libvirt_vm_engine` is `kvm`, otherwise `pc-1.0`.

    - `cpu_mode`: Virtual machine CPU mode. Default is `host-passthrough` if
      `libvirt_vm_engine` is `kvm`, otherwise `host-model`. Can be set to none
      to not configure a cpu mode.

    - `clock_offset`: Overrides default set in `libvirt_vm_clock_offset`

    - `enable_vnc`: If true enables VNC listening on localhost for use with
      VirtManager and similar tools

    - `volumes`: a list of volumes to attach to the VM.  Each volume is
      defined with the following dict:
        - `pool`: Name or UUID of the storage pool from which the volume should be
          allocated.
        - `name`: Name to associate with the volume being created; For `file` type volumes include extension if you would like volumes created with one.
        - `file_path`: Where the image of `file` type volumes should be placed; defaults to `libvirt_volume_default_images_path`
        - `device`: `disk` or `cdrom`
        - `capacity`: volume capacity (can be suffixed with M,G,T or MB,GB,TB, etc) (required when type is `disk`)
        - `format`: options include `raw`, `qcow2`, `vmdk`.  See `man virsh` for the
          full range.  Default is `qcow2`.
        - `image`: (optional) a URL to an image with which the volume is initalised (full copy).
        - `backing_image`: (optional) name of the backing volume which is assumed to already be the same pool (copy-on-write).
        - `image` and `backing_image` are mutually exclusive options.
        - `target`: (optional) Manually influence type and order of volumes

    - `interfaces`: a list of network interfaces to attach to the VM.
      Each network interface is defined with the following dict:

        - `type`: The type of the interface. Possible values:

            - `network`: Attaches the interface to a named Libvirt virtual
              network. This is the default value.
            - `direct`: Directly attaches the interface to one of the host's
              physical interfaces, using the `macvtap` driver.
        - `network`: Name of the network to which an interface should be
          attached. Must be specified if and only if the interface `type` is
          `network`.
        - `mac`: "Hardware" address of the virtual instance, if absent one is created
        - `source`: A dict defining the host interface to which this
          VM interface should be attached. Must be specified if and only if the
          interface `type` is `direct`. Includes the following attributes:

            - `dev`: The name of the host interface to which this VM interface
              should be attached.
            - `mode`: options include `vepa`, `bridge`, `private` and
              `passthrough`. See `man virsh` for more details. Default is
              `vepa`.

    - `console_log_enabled`: if `true`, log console output to a file at the
      path specified by `console_log_path`, **instead of** to a PTY. If
      `false`, direct terminal output to a PTY at serial port 0. Default is
      `false`.

    - `console_log_path`: Path to console log file. Default is
      `{{ libvirt_vm_default_console_log_dir }}/{{ name }}-console.log`.

    - `start`: Whether to immediately start the VM after defining it. Default
      is `true`.

    - `autostart`: Whether to start the VM when the host starts up. Default is
      `true`.

    - `xml_file`: Optionally supply a modified XML template. Base customisation
      off the default `vm.xml.j2` template so as to include the expected jinja
      expressions the role uses.

N.B. the following variables are deprecated: `libvirt_vm_state`,
`libvirt_vm_name`, `libvirt_vm_memory_mb`, `libvirt_vm_vcpus`,
`libvirt_vm_engine`, `libvirt_vm_machine`, `libvirt_vm_cpu_mode`,
`libvirt_vm_volumes`, `libvirt_vm_interfaces` and
`libvirt_vm_console_log_path`. If the variable `libvirt_vms` is left unset, its
default value will be a singleton list containing a VM specification using
these deprecated variables.

Dependencies
------------

If using qcow2 format drives qemu-img (in qemu-utils package) is required.


Example Playbook
----------------

    ---
    - name: Create VMs
      hosts: hypervisor
      roles:
        - role: stackhpc.libvirt-vm
          libvirt_vms:
            - state: present
              name: 'vm1'
              memory_mb: 512
              vcpus: 2
              volumes:
                - name: 'data1'
                  device: 'disk'
                  format: 'qcow2'
                  capacity: '400GB'
                  pool: 'my-pool'
                - name: 'debian-10.2.0-amd64-netinst.iso'
                  type: 'file'
                  device: 'cdrom'
                  format: 'raw'
                  target: 'hda'  # first device on ide bus
              interfaces:
                - network: 'br-datacentre'

            - state: present
              name: 'vm2'
              memory_mb: 1024
              vcpus: 1
              volumes:
                - name: 'data2'
                  device: 'disk'
                  format: 'qcow2'
                  capacity: '200GB'
                  pool: 'my-pool'
                - name: 'filestore'
                  type: 'file'
                  file_path: '/srv/cloud/images'
                  capacity: '900GB'
              interfaces:
                - type: 'direct'
                  source:
                    dev: 'eth123'
                    mode: 'private'
                - type: 'bridge'
                  source:
                    dev: 'br-datacentre'
