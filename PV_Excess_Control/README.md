# PV Excess Control

Automatically control your appliances (wallbox, heatpump, washing machine, ...) based on excess solar power.

## Features

:white*check_mark: Works with hybrid and standard inverters\
:white*check_mark: Works with hybrid installations that does not inject power onto the grid (zero feed in)
:white_check_mark: Configurable priority handling between multiple appliances\
:white_check_mark: Include solar forecasts from **Solcast** to ensure your home battery is charged to a specific level at the end of the day\
:white_check_mark: Battery charge logic considers hitorical energy consumption and optional appliance(s) consumption(s)s;
:white_check_mark: Define an \_On/Off switch interval* / solar power averaging interval\
:white*check_mark: Supports dynamic current control (e.g. for wallboxes)\
:white_check_mark: Define min. and max. current for appliances supporting dynamic current control\
:white_check_mark: Supports one- and three-phase appliances\
:white_check_mark: Supports \_Only-Switch-On\* devices like washing machines or dishwashers

## Prerequisites

- A working installation of [pyscript](https://github.com/custom-components/pyscript) (can be installed via [HACS](https://hacs.xyz/))
- (_Optional:_ A working installation of solcast (can be installed via [HACS custom repository](https://github.com/BJReplay/ha-solcast-solar)))
- Home Assistant v2023.1 or greater
- Access to the following values from your hybrid PV inverter:
  - Export power
  - PV Power
  - Load Power
  - Home battery level
- OR: Access to the following values from your standard inverter:
  - Combined import/export power
  - PV Power
- Pyscript must be configured to allow all imports. This can be done
  - either via UI:
    - Configuration -> Integrations page -> “+” -> Pyscript Python scripting
    - After that, you can change the settings anytime by selecting Options under Pyscript in the Configuration page
  - or via _`configuration.yaml`_:
    ```
    pyscript:
      allow_all_imports: true
    ```

## Installation

- Download (or clone) this GitHub repository
- Copy both folders (_blueprints_ and _pyscript_) to your HA config directory, or manually place the automation blueprint **`pv_excess_control.yaml`** and the python module **`pv_excess_control.py`** into their respective folders.
- Configure the desired logging level in your _`configuration.yaml`_:
  ```
  logger:
    logs:
      custom_components.pyscript.file.pv_excess_control: debug
  ```

## Configuration & Usage

### Initial Configuration

- For each appliance which should be controlled, create a new automation based on the _PV Excess Control_ blueprint
- After creating the automation, manually execute it once. This will send the chosen configuration parameters and sensors to the python module and start the optimizer in the background
- The python module stays active in background, even if HA or the complete system is restarted

### Zero feed in option

- Installations where hybrid inverters does not inject energy onto the grid\*\* (for ex. Growatt SPF series) have a particular condition where once the battery is fully charged, the inverter has to diminish solar power production to match the current energy load.
- This situation is tricky as the normal logic cannot detect excess of power to control optional loads.
- The zero feed in option attempts to detect the condition and enable appliances after the battery charge threshold is archived (Zero Feed In - Battery Level). It does this by changing decection logic and relying on solar forecast production, therefore a working installation of solcast is required.

\*\*The condition does not happen with installations injecting energy onto the grid, as once the battery is full they should start exporting energy and it is detected by the automation.

### Home battery charging

The logic prioritizes the best it can to have battery charged to the threshold level set by the end of the day.
Given changes in energy load, solar forecast or appliance demands(ex appliance minimum daily runtime) the desired threshold might not be archived.

However, under a fully working and tuned setup, the automation is almost always able to reach desired battery charge with a margin of 5-10% error.

### Update

- To update the configuration, simply update the chosen parameters and values in your automation, which was created based on the blueprint.
- After that, manually execute the automation once to send the changes to the python module

### Deactivation

- To deactivate the auto-control of a single appliance, simply deactivate the related automation.

### Deletion

- To remove the auto-control of a single appliance, simply delete the related automation.

## Credits

Originally based and created by https://github.com/InventoCasa/ha-advanced-blueprints
