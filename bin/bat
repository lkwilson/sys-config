#!/usr/bin/env python

def get(prop):
  base_path = '/sys/class/power_supply/BAT0/'
  with open(base_path + prop) as fin:
    contents = fin.read()
  return contents.strip()

def main():
  status = get('status') # str
  manufacturer = get('manufacturer') # str
  model_name = get('model_name') # str
  serial_number = get('serial_number') # str

  # capacity
  capacity_level = get('capacity_level') # str
  capacity = int(get('capacity')) # %

  # charge, Ah
  charge_now = int(get('charge_now'))/1e6
  charge_full = int(get('charge_full'))/1e6
  charge_full_design = int(get('charge_full_design'))/1e6

  # current, A
  current_now = int(get('current_now'))/1e6

  # volts, V
  voltage_min_design = int(get('voltage_min_design'))/1e6
  voltage_now = int(get('voltage_now'))/1e6

  # power, W
  power_now = voltage_now * current_now

  print('Current:')
  print(f'{capacity_level} ({capacity}%, {status})')
  print(f'Charge: {charge_now}/{charge_full} ({charge_now/charge_full*100:.1f}%) Ah')
  print(f'Voltage: {voltage_now} (min:{voltage_min_design}, delta:{voltage_now - voltage_min_design:.6f}) V')
  print(f'Current: {current_now} A')
  print(f'Power: {power_now:.6} W')
  print()
  print('Health:')
  print('Manufacturer:', manufacturer)
  print('Model Name:', model_name)
  print('Serial Number:', serial_number)
  print(f'Voltage: {voltage_now} (min:{voltage_min_design}, delta:{voltage_now - voltage_min_design:.6f}) V')
  print(f'Battery: {charge_full}/{charge_full_design} ({charge_full/charge_full_design*100:.1f}%) Ah')

if __name__ == '__main__':
  main()

