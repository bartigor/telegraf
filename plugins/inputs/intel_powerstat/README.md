# Intel PowerStat Input Plugin

This input plugin monitors power statistics on Intel-based platforms and assumes presence of Linux based OS.

Main use cases are power saving and workload migration. Telemetry frameworks allow users to monitor critical platform level metrics.
Key source of platform telemetry is power domain that is beneficial for MANO/Monitoring&Analytics systems
to take preventive/corrective actions based on platform busyness, CPU temperature, actual CPU utilization and power statistics.

## Configuration

```toml @sample.conf
# Intel PowerStat plugin enables monitoring of platform metrics (power, TDP) and per-CPU metrics like temperature, power and utilization.
[[inputs.intel_powerstat]]
  ## The user can choose which package metrics are monitored by the plugin with the package_metrics setting:
  ## - The default, will collect "current_power_consumption", "current_dram_power_consumption" and "thermal_design_power"
  ## - Setting this value to an empty array means no package metrics will be collected
  ## - Finally, a user can specify individual metrics to capture from the supported options list
  ## Supported options:
  ##   "current_power_consumption", "current_dram_power_consumption", "thermal_design_power", "max_turbo_frequency"
  # package_metrics = ["current_power_consumption", "current_dram_power_consumption", "thermal_design_power"]

  ## The user can choose which per-CPU metrics are monitored by the plugin in cpu_metrics array.
  ## Empty or missing array means no per-CPU specific metrics will be collected by the plugin.
  ## Supported options:
  ##   "cpu_frequency", "cpu_c0_state_residency", "cpu_c1_state_residency", "cpu_c6_state_residency", "cpu_busy_cycles", "cpu_temperature", "cpu_busy_frequency"
  ## ATTENTION: cpu_busy_cycles option is DEPRECATED - superseded by cpu_c0_state_residency
  # cpu_metrics = []
```

## Example: Configuration with no per-CPU telemetry

This configuration allows getting default processor package specific metrics, no per-CPU metrics are collected:

```toml
[[inputs.intel_powerstat]]
  cpu_metrics = []
```

## Example: Configuration with no per-CPU telemetry - equivalent case

This configuration allows getting default processor package specific metrics, no per-CPU metrics are collected:

```toml
[[inputs.intel_powerstat]]
```

## Example: Configuration for CPU Temperature and CPU Frequency

This configuration allows getting default processor package specific metrics, plus subset of per-CPU metrics (CPU Temperature and CPU Frequency):

```toml
[[inputs.intel_powerstat]]
  cpu_metrics = ["cpu_frequency", "cpu_temperature"]
```

## Example: Configuration for CPU Temperature and CPU Frequency without default package metrics

This configuration allows getting only a subset of per-CPU metrics (CPU Temperature and CPU Frequency):

```toml
[[inputs.intel_powerstat]]
  package_metrics = []
  cpu_metrics = ["cpu_frequency", "cpu_temperature"]
```

## Example: Configuration with all available metrics

This configuration allows getting all processor package specific metrics and all per-CPU metrics:

```toml
[[inputs.intel_powerstat]]
  package_metrics = ["current_power_consumption", "current_dram_power_consumption", "thermal_design_power", "max_turbo_frequency"]
  cpu_metrics = ["cpu_frequency", "cpu_busy_frequency", "cpu_temperature", "cpu_c0_state_residency", "cpu_c1_state_residency", "cpu_c6_state_residency"]
```

## SW Dependencies

Plugin is based on Linux Kernel modules that expose specific metrics over `sysfs` or `devfs` interfaces.
The following dependencies are expected by plugin:

- _intel-rapl_ module which exposes Intel Runtime Power Limiting metrics over `sysfs` (`/sys/devices/virtual/powercap/intel-rapl`),
- _msr_ kernel module that provides access to processor model specific registers over `devfs` (`/dev/cpu/cpu%d/msr`),
- _cpufreq_ kernel module - which exposes per-CPU Frequency over `sysfs` (`/sys/devices/system/cpu/cpu%d/cpufreq/scaling_cur_freq`).

Minimum kernel version required is 3.13 to satisfy all requirements.

Please make sure that kernel modules are loaded and running (cpufreq is integrated in kernel). Modules might have to be manually enabled by using `modprobe`.
Depending on the kernel version, run commands:

```sh
# kernel 5.x.x:
sudo modprobe rapl
subo modprobe msr
sudo modprobe intel_rapl_common
sudo modprobe intel_rapl_msr

# kernel 4.x.x:
sudo modprobe msr
sudo modprobe intel_rapl
```

**Telegraf with Intel PowerStat plugin enabled may require root access to read model specific registers (MSRs)**
to retrieve data for calculation of most critical per-CPU specific metrics:

- `cpu_busy_frequency_mhz`
- `cpu_temperature_celsius`
- `cpu_c0_state_residency_percent`
- `cpu_c1_state_residency_percent`
- `cpu_c6_state_residency_percent`

and to retrieve data for calculation per-package specific metric:

- `max_turbo_frequency_mhz`

To expose other Intel PowerStat metrics root access may or may not be required (depending on OS type or configuration).

## HW Dependencies

Specific metrics require certain processor features to be present, otherwise Intel PowerStat plugin won't be able to
read them. When using Linux Kernel based OS, user can detect supported processor features reading `/proc/cpuinfo` file.
Plugin assumes crucial properties are the same for all CPU cores in the system.
The following processor properties are examined in more detail in this section:
processor _cpu family_, _model_ and _flags_.
The following processor properties are required by the plugin:

- Processor _cpu family_ must be Intel (0x6) - since data used by the plugin assumes Intel specific
model specific registers for all features
- The following processor flags shall be present:
  - "_msr_" shall be present for plugin to read platform data from processor model specific registers and collect
    the following metrics: _powerstat\_core.cpu\_temperature_, _powerstat\_core.cpu\_busy\_frequency_,
    _powerstat\_core.cpu\_c0\_state\_residency_, _powerstat\_core.cpu\_c1\_state\_residency_, _powerstat\_core.cpu\_c6\_state\_residency_
  - "_aperfmperf_" shall be present to collect the following metrics: _powerstat\_core.cpu\_busy\_frequency_,
    _powerstat\_core.cpu\_c0\_state\_residency_, _powerstat\_core.cpu\_c1\_state\_residency_
  - "_dts_" shall be present to collect _powerstat\_core.cpu\_temperature_
- Processor _Model number_ must be one of the following values for plugin to read _powerstat\_core.cpu\_c1\_state\_residency_
and _powerstat\_core.cpu\_c6\_state\_residency_ metrics:

| Model number | Processor name |
|-----|-------------|
| 0x37 | Intel Atom® Bay Trail |
| 0x4D | Intel Atom® Avaton |
| 0x5C | Intel Atom® Apollo Lake |
| 0x5F | Intel Atom® Denverton |
| 0x7A | Intel Atom® Goldmont |
| 0x4C | Intel Atom® Airmont |
| 0x86 | Intel Atom® Jacobsville |
| 0x96 | Intel Atom® Elkhart Lake |
| 0x9C | Intel Atom® Jasper Lake |
| 0x1A | Intel Nehalem-EP |
| 0x1E | Intel Nehalem |
| 0x1F | Intel Nehalem-G |
| 0x2E | Intel Nehalem-EX |
| 0x25 | Intel Westmere |
| 0x2C | Intel Westmere-EP |
| 0x2F | Intel Westmere-EX |
| 0x2A | Intel Sandybridge |
| 0x2D | Intel Sandybridge-X |
| 0x3A | Intel Ivybridge |
| 0x3E | Intel Ivybridge-X |
| 0x4E | Intel Atom® Silvermont-MID |
| 0x5E | Intel Skylake |
| 0x55 | Intel Skylake-X |
| 0x8E | Intel Kabylake-L |
| 0x9E | Intel Kabylake |
| 0x6A | Intel Icelake-X |
| 0x6C | Intel Icelake-D |
| 0x7D | Intel Icelake |
| 0x7E | Intel Icelake-L |
| 0x9D | Intel Icelake-NNPI |
| 0x3C | Intel Haswell |
| 0x3F | Intel Haswell-X |
| 0x45 | Intel Haswell-L |
| 0x46 | Intel Haswell-G |
| 0x3D | Intel Broadwell |
| 0x47 | Intel Broadwell-G |
| 0x4F | Intel Broadwell-X |
| 0x56 | Intel Broadwell-D |
| 0x66 | Intel Cannonlake-L |
| 0x57 | Intel Xeon® PHI Knights Landing |
| 0x85 | Intel Xeon® PHI Knights Mill |
| 0xA5 | Intel CometLake |
| 0xA6 | Intel CometLake-L |
| 0x8F | Intel Sapphire Rapids X |
| 0x8C | Intel TigerLake-L |
| 0x8D | Intel TigerLake |

## Metrics

All metrics collected by Intel PowerStat plugin are collected in fixed intervals.
Metrics that reports processor C-state residency or power are calculated over elapsed intervals.
When starting to measure metrics, plugin skips first iteration of metrics if they are based on deltas with previous value.

**The following measurements are supported by Intel PowerStat plugin:**

- powerstat_core

  - The following Tags are returned by plugin with powerstat_core measurements:

      | Tag          | Description                   |
      |--------------|-------------------------------|
      | `package_id` | ID of platform package/socket |
      | `core_id`    | ID of physical processor core |
      | `cpu_id`     | ID of logical processor core  |

   Measurement powerstat_core metrics are collected per-CPU (cpu_id is the key)
   while core_id and package_id tags are additional topology information.

  - Available metrics for powerstat_core measurement
  
      | Metric name (field) | Description | Units |
      |---------------------|-------------|-------|
      | `cpu_frequency_mhz` | Current operational frequency of CPU Core | MHz |
      | `cpu_busy_frequency_mhz`  | CPU Core Busy Frequency measured as frequency adjusted to CPU Core busy cycles | MHz |
      | `cpu_temperature_celsius` | Current temperature of CPU Core | Celsius degrees |
      | `cpu_c0_state_residency_percent` | Percentage of time that CPU Core spent in C0 Core residency state | % |
      | `cpu_c1_state_residency_percent` | Percentage of time that CPU Core spent in C1 Core residency state | % |
      | `cpu_c6_state_residency_percent` | Percentage of time that CPU Core spent in C6 Core residency state | % |
      | `cpu_busy_cycles_percent` | (**DEPRECATED** - superseded by cpu_c0_state_residency_percent) CPU Core Busy cycles as a ratio of Cycles spent in C0 state residency to all cycles executed by CPU Core | % |

- powerstat_package

  - The following Tags are returned by plugin with powerstat_package measurements:

      | Tag | Description |
      |-----|-------------|
      | `package_id` | ID of platform package/socket |
      | `active_cores`| Specific tag for `max_turbo_frequency_mhz` metric. The maximum number of activated cores for reachable turbo frequency

   Measurement powerstat_package metrics are collected per processor package -_package_id_ tag indicates which package metric refers to.

  - Available metrics for powerstat_package measurement

      | Metric name (field) | Description | Units |
      |-----|-------------|-----|
      | `thermal_design_power_watts` |  Maximum Thermal Design Power (TDP) available for processor package | Watts |
      | `current_power_consumption_watts` | Current power consumption of processor package | Watts |
      | `current_dram_power_consumption_watts` | Current power consumption of processor package DRAM subsystem | Watts |
      | `max_turbo_frequency_mhz`| Maximum reachable turbo frequency for number of cores active | MHz

### Known issues

From linux kernel version v5.4.77 with [this kernel change](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?h=v5.4.77&id=19f6d91bdad42200aac557a683c17b1f65ee6c94)
resources like `/sys/class/powercap/intel-rapl*/*/energy_uj` are readable only by root for security reasons, so this plugin needs root privileges to work properly.

If such strict security restrictions are not relevant, reading permissions to files in `/sys/devices/virtual/powercap/intel-rapl/`
directory can be manually changed for example with `chmod` command with custom parameters.
For example to give all users permission to all files in `intel-rapl` directory:

```bash
sudo chmod -R a+rx /sys/devices/virtual/powercap/intel-rapl/
```

### Example Output

```shell
powerstat_package,host=ubuntu,package_id=0 thermal_design_power_watts=160 1606494744000000000
powerstat_package,host=ubuntu,package_id=0 current_power_consumption_watts=35 1606494744000000000
powerstat_package,host=ubuntu,package_id=0 current_dram_power_consumption_watts=13.94 1606494744000000000
powerstat_package,host=ubuntu,package_id=0,active_cores=0 max_turbo_frequency_mhz=3000i 1606494744000000000
powerstat_package,host=ubuntu,package_id=0,active_cores=1 max_turbo_frequency_mhz=2800i 1606494744000000000
powerstat_core,core_id=0,cpu_id=0,host=ubuntu,package_id=0 cpu_frequency_mhz=1200.29 1606494744000000000
powerstat_core,core_id=0,cpu_id=0,host=ubuntu,package_id=0 cpu_temperature_celsius=34i 1606494744000000000
powerstat_core,core_id=0,cpu_id=0,host=ubuntu,package_id=0 cpu_c6_state_residency_percent=92.52 1606494744000000000
powerstat_core,core_id=0,cpu_id=0,host=ubuntu,package_id=0 cpu_c1_state_residency_percent=6.68 1606494744000000000
powerstat_core,core_id=0,cpu_id=0,host=ubuntu,package_id=0 cpu_c0_state_residency_percent=0.8 1606494744000000000
powerstat_core,core_id=0,cpu_id=0,host=ubuntu,package_id=0 cpu_busy_frequency_mhz=1213.24 1606494744000000000
```
