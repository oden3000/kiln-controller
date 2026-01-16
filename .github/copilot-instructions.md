# Kiln Controller - AI Coding Instructions

## Project Overview
Kiln Controller is a Raspberry Pi-based ceramic kiln firing system with web interface control, real-time temperature monitoring via thermocouples, PID-based heating control, and firing profile support.

**Core Purpose**: Convert inexpensive hardware (RPi + thermocouple board + SSR relay) into a production kiln controller that supports repeatable, programmable kiln firings with safety monitoring.

## Architecture Overview

### Multi-Threaded Event Loop Design
- **Main server thread**: Bottle web framework (HTTP/WebSocket) on port 8081
- **Oven thread**: Independent control loop (reads sensor every 2s by default, computes PID, actuates relay)
- **OvenWatcher thread**: Aggregates oven state, broadcasts via WebSocket to all connected clients
- **Sensor threads**: Each thermocouple board reader is a daemon thread that samples temperature continuously

**Why separate threads?** Temperature control must run independently of web I/O to guarantee consistent timing for PID tuning (critical for kiln heating). Use `time.sleep()` in loops, not blocking I/O.

### Key Classes & Data Flow

#### `Oven` (lib/oven.py - Abstract base)
- **States**: `IDLE`, `RUNNING`, `PAUSED`
- **Key methods**: `run_profile(profile, startat=0)`, `abort_run()`, `get_state()`
- **Outputs**: `self.heat` (0.0-1.0 duty cycle), `self.temperature`, `self.target`
- **PID loop**: Every sensor_time_wait (default 2s), calls `heat_then_cool()` which uses PID to modulate heater relay

#### `SimulatedOven` vs `RealOven` 
- **SimulatedOven**: Uses physics model (thermal capacitance, heat transfer equations) with configurable speedup factor (default 10x)
- **RealOven**: Reads actual thermocouple via MAX31855/MAX31856, controls GPIO relay
- **Key difference**: SimulatedOven updates runtime with speedup factor; RealOven uses wall clock time

#### `Profile` (lib/oven.py)
- **Data format**: `{"name": "...", "data": [[seconds, temperature], ...], "temp_units": "c"}`
- Profiles stored in `storage/profiles/` as JSON; always stored internally in Celsius
- **Core methods**: `get_target_temperature(runtime_seconds)`, `find_next_time_from_temperature(temp)` 
- **Interpolation**: Linear interpolation between consecutive (time, temp) points
- **Scheduling**: Profiles can be skipped ahead (`startat` parameter) to match current kiln temp when user clicks run

#### `OvenWatcher` (lib/ovenWatcher.py)
- Observer pattern: Collects oven state every cycle, broadcasts JSON to all WebSocket clients
- **Key feature**: `lastlog_subset()` downsamples state history (keeps ~50 points for graphs)
- Clients only receive initial backlog of data on connect, then continuous updates

### Temperature Sensing Pipeline
1. **Hardware**: MAX31855 (K-type only) or MAX31856 (multiple thermocouple types)
2. **Blinka abstraction**: Auto-detects board type (RPi, BeagleBone, etc.) via `board.board_id`
3. **SPI**: Hardware SPI (fast) auto-selected; falls back to bitbang if pins configured
4. **Thermocouple errors**: Custom exception hierarchy maps Adafruit library errors to consistent messages (e.g., "not connected", "short circuit")
5. **Averaging**: Takes `temperature_average_samples` readings per duty cycle, uses **median** (not mean) for outlier rejection
6. **Offset**: Applied at read-time in `get_state()` via `config.thermocouple_offset`

### PID Control Flow
- **Setpoint**: Target temperature from profile
- **Input**: Actual kiln temperature (+ offset)
- **Output**: `pid` (0.0-1.0) that becomes duty cycle for relay
- **RealOven**: `heat_on = time_step * pid`, alternates heat/cool within each cycle
- **Key antiwindup**: `config.pid_control_window` - integral only accumulates when error < this threshold
- **Tracking**: `self.pid.pidstats` dict logged and sent to UI for tuning diagnosis

### Web Architecture
**Four WebSocket endpoints** (no REST, all persistent connections):
- `/status`: OvenWatcher broadcasts oven state (temp, heat, runtime, cost) every cycle
- `/control`: Receives RUN/STOP/SIMULATE commands, initiates profile execution
- `/storage`: Manages profile CRUD (GET/PUT/DELETE)
- `/config`: Sends config values (temp_scale, costs, time formats)

**Static files**: `/picoreflow/` route serves `public/` directory
- Index pulls UI from `public/index.html` and `public/assets/js/picoreflow.js`
- Graphs via Flot jQuery plugin, real-time updates via WebSocket

### Config System
**All constants in `config.py`** (no env vars):
- **Hardware pins**: GPIO assignments for relay, thermocouple CS/CLK/MISO/MOSI
- **PID tuning**: `pid_kp`, `pid_ki` (inverted - smaller = more integral), `pid_kd`
- **Simulation**: `simulate=True/False`, physics constants (`sim_c_heat`, `sim_R_ho`, speedup_factor)
- **Safety thresholds**: `emergency_shutoff_temp`, `ignore_tc_*` flags for error suppression
- **Cost calculation**: `kw_elements`, `kwh_rate`, `currency_type`
- **Temperature conversion**: Always store profiles in Celsius; normalize on load/save if user's scale is Fahrenheit

## Key Patterns & Conventions

### Threading & Timing
```python
# DO: Use config.sensor_time_wait (2s by default) as duty cycle
time.sleep(config.sensor_time_wait)

# DON'T: Use sleep() without reason - breaks PID timing consistency
# DON'T: Block I/O in the Oven thread
```

### Error Handling: Thermocouple-Specific
- Adafruit library doesn't throw exceptions for some faults (MAX31856)
- Check `self.thermocouple.fault` dict explicitly
- Map errors to consistent messages via `ThermocoupleError.map` dict
- Use `config.ignore_tc_*` flags to selectively suppress errors (e.g., ignore disconnects during idle)

### State Management
- **Oven.state** drives main loop behavior; only three valid values
- **Call `reset()` before `run_profile()`** to clear cost, runtime, etc.
- **Automatic restart**: Saved state serialized to `config.automatic_restart_state_file` every cycle; restored on boot if profile was RUNNING

### Profile Interpolation
```python
# Linear interpolation between consecutive points
# If profile.data = [(0, 20), (3600, 200), (7200, 200)]:
#   at 1800s -> 110°C (halfway from 20 to 200)
#   at 5400s -> 200°C (on flat segment)
# get_surrounding_points() always returns prev/next pair; handles edge cases
```

### Temperature Unit Conversion
- **Internal storage**: Always Celsius in Profile.data
- **On display**: Convert if `config.temp_scale == "f"`
- **On save**: Profile.temp_units stored; `normalize_temp_units()` called on load to ensure UI gets correct scale

### WebSocket Message Formats
**OvenWatcher broadcast** (status):
```json
{
  "temperature": 105.5,
  "target": 150.0,
  "state": "RUNNING",
  "runtime": 1234.5,
  "heat": 0.75,
  "cost": 0.42,
  "pidstats": {"p": 15.0, "i": 2.1, "d": 5.3, "pid": 22.4}
}
```

**Control command** (from UI):
```json
{"cmd": "RUN", "profile": {...}, "startat": 0}
```

### Simulation vs Real Execution
- **Simulation** (`config.simulate = True`): Uses `SimulatedOven`, physics model advances with `speedup_factor`
- **Real** (`config.simulate = False`): Uses `RealOven`, wall-clock timing, GPIO control
- **Testing**: Tests can run against SimulatedOven with fast speedup (e.g., 60x); no hardware required

## Critical Developer Workflows

### Running the App
```bash
python3 -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r requirements.txt
python3 kiln-controller.py
# Browse to http://localhost:8081
```

### Testing Framework & Bug Reproduction

#### Unit Tests (Profile Logic)
Profile interpolation and scheduling calculations use pytest. Run with:
```bash
pytest Test/test_Profile.py -v
```

**What's tested**: 
- `get_target_temperature(runtime)` - linear interpolation between profile points
- `find_next_time_from_temperature(temp)` - reverse lookup for seek-start feature
- `find_x_given_y_on_line_from_two_points()` - helper for interpolation edge cases

**Test profiles available**:
- `Test/test-fast.json` - Simple 6-point profile for basic tests
- `Test/test-cases.json` - Edge cases: flat segments, multiple temperature ranges

#### Integration Testing via Simulation (Key for Bug Reproduction)
Enable simulation mode in `config.py` to test without hardware:

```python
# config.py
simulate = True                    # Use SimulatedOven physics model
sim_speedup_factor = 10           # 10x real-time (faster testing)
# Optional: adjust physics for different kilns
sim_p_heat = 5450.0               # Element wattage
sim_c_oven = 5000.0               # Thermal mass
sim_R_ho_noair = 0.1              # Heat transfer resistance
```

**Run full firing simulation**:
1. Start controller: `python3 kiln-controller.py`
2. Open http://localhost:8081 in browser
3. Select a profile, click "Run"
4. Watch logs in terminal for timing, PID values, temperature progression
5. Check cost calculation, heat rate, state transitions

**Example bug reproduction workflow**:
- Reproduce temperature overshoot: Lower `sim_speedup_factor` to 5, increase `pid_ki` aggressiveness
- Test seek-start edge case: Load profile starting at 20°C, run when simulated kiln is already at 500°C
- Profile interpolation bug: Create test profile with steep ramps, verify `get_target_temperature()` at specific times
- Cost tracking: Run full profile, check final cost calculation in logs

#### Hardware-Specific Component Tests
Isolated tests for non-simulation hardware (RPi/BeagleBone required):

**Relay/GPIO output test**:
```bash
python3 test-output.py
```
Toggles `config.gpio_heat` every 5 seconds. Monitor with:
```bash
python3 gpioreadall.py  # in another terminal
```

**Thermocouple sensor test**:
```bash
python3 test-thermocouple.py
```
Reads temperature every 1 second. Configure SPI pins in `config.py` first. Touch thermocouple to verify response.

#### Debugging Real Firing Issues
When reproducing bugs found on actual kiln:

1. **Capture logs**:
   ```bash
   python3 kiln-controller.py 2>&1 | tee kiln-run.log
   # Run your firing, then Ctrl+C
   ```

2. **Enable verbose logging** in `config.py`:
   ```python
   log_level = logging.DEBUG  # (change from logging.INFO)
   ```

3. **Create minimal reproducible profile**:
   ```json
   {
     "name": "bug-repro",
     "data": [[0, 70], [300, 100], [600, 100], [900, 70]],
     "temp_units": "c"
   }
   ```

4. **Test in simulation first**:
   - Set `simulate = True` with same PID params as real kiln
   - Run bug-repro profile at various `sim_speedup_factor` values
   - Compare simulated PID output against real kiln behavior

5. **Extract PID stats** from logs:
   ```
   grep "temp=.*target=.*pid=" kiln-run.log | head -20
   ```
   Look for patterns: integral buildup, derivative spikes, oscillation

#### Creating Custom Test Profiles
Test profiles are stored in `Test/test-*.json`. Create new ones:

```json
{
  "name": "test-my-bug",
  "data": [
    [0, 70],           // Start at room temp
    [600, 200],        // 10min ramp to 200°C (test slow ramp)
    [1200, 200],       // Hold 10min (test integral windup)
    [1800, 500],       // Rapid climb (test derivative)
    [2400, 500],       // Soak (test overshoot recovery)
    [3000, 70]         // Cool down
  ],
  "temp_units": "c"
}
```

Add test case to `Test/test_Profile.py`:
```python
def test_my_bug_ramp_rate():
    profile = get_profile("test-my-bug.json")
    # At 900s (halfway through slow ramp)
    temp = profile.get_target_temperature(900)
    assert 134 < temp < 136  # Should be ~135°C
```

### PID Tuning Workflow
1. Enable `config.simulate = True` with realistic physics constants
2. Run a test profile at 10x speedup, monitor `pidstats` in UI
3. Adjust `pid_kp`, `pid_ki`, `pid_kd` in config, restart to see effect
4. On real hardware: smaller adjustments recommended, tune slowly

### Adding a New Thermocouple Type
1. Subclass `TempSensorReal` (see `Max31856` example)
2. Implement `raw_temp()` to read and validate sensor
3. Raise custom error subclass with fault mapping
4. Add import & selection logic in `RealBoard.choose_tempsensor()`
5. Update `config.py` documentation

## Critical Integration Points

### External Dependencies
- **Bottle**: HTTP/WebSocket server (minimal, intentional choice)
- **Gevent**: Async I/O for WebSocket handling
- **Adafruit Blinka**: GPIO abstraction across Raspberry Pi, BeagleBone, etc.
- **Adafruit MAX31855/MAX31856**: Thermocouple chip libraries (pure Python I2C/SPI)

### GPIO Control (RealOven only)
- Uses `digitalio.DigitalInOut` (Blinka abstraction)
- Single pin controls SSR via transistor buffer (Blinka handles voltage negotiation)
- `gpio_heat_invert` flag: some SSRs active-low; config handles polarity

### Logging
- **All modules** use `logging.getLogger(__name__)`
- **Deduplication**: `DupFilter` prevents same message flooding logs during errors
- **Config-driven**: `config.log_level` and `config.log_format` control verbosity

## Discoverable Patterns (From Inspection)

### Seek-Start Feature
- User clicks "Run" when kiln is already hot → `config.seek_start = True` triggers auto-skip
- `Oven.get_start_from_temperature(profile, temp)` finds first profile point above current temp
- Prevents long flat ramping if profile starts at room temp but kiln is at 800°C

### Cost Tracking
- Updated every cycle: `cost += (kwh_rate * kw_elements) * (heat_on_time / 3600)`
- Displayed in real-time to user; shown in logs at end of run
- Useful for comparing profile efficiency

### Catching-Up Logic
- If kiln temperature drifts > `pid_control_window` from setpoint, shift schedule forward to wait
- Prevents integral windup by pausing profile progression until kiln stabilizes
- Flag `catching_up` sent to UI to signal user of delay

## Common Gotchas & Mitigations

1. **Profile format mismatch**: Test always use `test_Profile.py` which expects data as sorted list of tuples
2. **GPIO permissions**: RealOven requires sudo or GPIO group membership; test with simulate=True first
3. **Temperature unit confusion**: Always normalize profiles on load; config.temp_scale controls display only
4. **PID integral windup**: Off by design with `pid_control_window` threshold; tuning notes in docs/pid_tuning.md
5. **Thermocouple disconnects**: Each error type configurable to ignore/abort; log errors even when ignored for debugging
