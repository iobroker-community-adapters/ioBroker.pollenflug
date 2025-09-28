# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### Pollenflug Adapter Context

This is the **pollenflug** adapter that provides pollen flight risk index data from the German Weather Service (DWD). Key characteristics:

- **Primary Function**: Fetches pollen risk indices from DWD OpenData API for German regions
- **Data Source**: https://opendata.dwd.de/climate_environment/health/alerts/s31fg.json
- **Target Users**: People with pollen allergies who need daily pollen forecasts
- **Data Types**: Pollen risk indices for various species (hazel, alder, ash, birch, grass, rye, mugwort, ambrosia)
- **Update Schedule**: Daily updates around 11 AM from DWD
- **Regional Coverage**: All German federal states with specific region codes
- **Forecast Period**: Today, tomorrow, and day after tomorrow (Friday forecasts include Sunday)

#### Technical Implementation Details:
- Uses XML/JSON parsing for DWD data processing
- Implements scheduled data fetching with configurable intervals
- Stores risk indices as ioBroker states per pollen type and region
- Provides validity period information for each forecast
- Supports multiple German regions via region code selection

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Check for essential states
                        const states = await harness.states.getStatesAsync('your-adapter.0.*');
                        const stateCount = Object.keys(states).length;
                        
                        console.log(`Found ${stateCount} states`);
                        if (stateCount === 0) {
                            return reject(new Error('No states created - adapter may not be working correctly'));
                        }

                        console.log('✅ Integration test completed successfully');
                        resolve(true);
                        
                    } catch (error) {
                        console.error('❌ Integration test failed:', error.message);
                        reject(error);
                    }
                });
            });
        });
    }
});
```

#### Additional Test Requirements for API-dependent Adapters

For adapters like this one that depend on external APIs, create separate test suites:

1. **Unit Tests**: Test internal logic with mocked API responses
2. **Integration Tests**: Test adapter initialization and state management
3. **API Tests**: Optional tests that verify API connectivity (should not be required for CI/CD)

Example API test structure for pollenflug adapter:
```javascript
// test/integration-api.js (optional, for manual testing)
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('API Connectivity Test', (getHarness) => {
            it('should fetch data from DWD API', async function() {
                // Test actual API connectivity
                // This should be optional and not block CI/CD
            });
        });
    }
});
```

### Testing Best Practices
- Always mock external dependencies in unit tests
- Use realistic test data that matches the expected API responses
- Test error conditions (network failures, invalid responses, etc.)
- Ensure tests can run without internet connectivity
- For pollenflug: Create mock DWD JSON responses for comprehensive testing

## ioBroker Integration Patterns

### Adapter Lifecycle Methods

#### Main Adapter Class Pattern
```javascript
class PollenflugAdapter extends utils.Adapter {
    constructor(options) {
        super({ ...options, name: 'pollenflug' });
        this.on('ready', this.onReady.bind(this));
        this.on('stateChange', this.onStateChange.bind(this));
        this.on('objectChange', this.onObjectChange.bind(this));
        this.on('message', this.onMessage.bind(this));
        this.on('unload', this.onUnload.bind(this));
    }

    async onReady() {
        // Initialize adapter
        this.setState('info.connection', false, true);
        
        // Load configuration
        const config = this.config;
        
        // Setup scheduled tasks
        this.scheduleTask();
    }

    onUnload(callback) {
        try {
            // Clean up resources
            if (this.timeout) {
                clearTimeout(this.timeout);
            }
            if (this.interval) {
                clearInterval(this.interval);  
            }
            callback();
        } catch (e) {
            callback();
        }
    }
}
```

### State Management

#### State Creation and Updates
```javascript
// Create objects for pollen types
const pollenTypes = ['hazel', 'alder', 'ash', 'birch', 'grass', 'rye', 'mugwort', 'ambrosia'];

for (const pollen of pollenTypes) {
    await this.setObjectNotExistsAsync(`today.${pollen}`, {
        type: 'state',
        common: {
            name: `Pollen risk ${pollen} today`,
            type: 'number',
            role: 'value',
            read: true,
            write: false,
            min: 0,
            max: 3,
            states: {
                0: 'no risk',
                1: 'low risk', 
                2: 'medium risk',
                3: 'high risk'
            }
        },
        native: {}
    });
}

// Update state values
await this.setStateAsync('today.birch', { val: riskLevel, ack: true });
```

#### Connection Status Management
```javascript
// Always maintain connection status
this.setState('info.connection', true, true);  // Connected
this.setState('info.connection', false, true); // Disconnected
```

### Configuration Management

#### Reading Configuration
```javascript
onReady() {
    const pollInterval = this.config.pollInterval || 5;
    const region = this.config.region || '*';
    const url = this.config.url;
    const sentryEnabled = this.config.sentry_enable;
    
    this.log.info(`Configuration: interval=${pollInterval}, region=${region}`);
}
```

#### Configuration Validation
```javascript
validateConfig() {
    if (!this.config.url) {
        this.log.error('No URL configured');
        return false;
    }
    
    if (!this.config.region || this.config.region === '*') {
        this.log.warn('No specific region selected, using default');
    }
    
    return true;
}
```

### Data Fetching and Processing

#### HTTP Requests with Error Handling
```javascript
const request = require('request-promise-native');

async fetchPollenData() {
    try {
        this.log.debug('Fetching pollen data from DWD...');
        
        const options = {
            uri: this.config.url,
            json: true,
            timeout: 10000
        };
        
        const response = await request(options);
        
        if (!response) {
            throw new Error('Empty response from API');
        }
        
        await this.processPollenData(response);
        this.setState('info.connection', true, true);
        
    } catch (error) {
        this.log.error(`Failed to fetch pollen data: ${error.message}`);
        this.setState('info.connection', false, true);
    }
}
```

#### Data Processing Pattern
```javascript
async processPollenData(data) {
    try {
        // Process the DWD data format
        const regions = data.content || [];
        const targetRegion = this.config.region;
        
        for (const region of regions) {
            if (targetRegion === '*' || region.region_id === targetRegion) {
                await this.updatePollenStates(region);
            }
        }
        
        this.log.info('Pollen data processed successfully');
        
    } catch (error) {
        this.log.error(`Error processing pollen data: ${error.message}`);
    }
}
```

### Scheduling and Intervals

#### Task Scheduling
```javascript
scheduleTask() {
    // Clear existing schedule
    if (this.pollTimeout) {
        clearTimeout(this.pollTimeout);
    }
    
    // Schedule next update
    const pollInterval = this.config.pollInterval || 60; // minutes
    const intervalMs = pollInterval * 60 * 1000;
    
    this.pollTimeout = setTimeout(() => {
        this.fetchPollenData();
        this.scheduleTask(); // Reschedule
    }, intervalMs);
    
    this.log.debug(`Next poll scheduled in ${pollInterval} minutes`);
}
```

### Error Handling

#### Comprehensive Error Handling
```javascript
async main() {
    try {
        if (!this.validateConfig()) {
            return;
        }
        
        await this.initializeObjects();
        await this.fetchPollenData();
        this.scheduleTask();
        
    } catch (error) {
        this.log.error(`Adapter initialization failed: ${error.message}`);
        this.setState('info.connection', false, true);
    }
}

// Proper cleanup
onUnload(callback) {
  try {
    this.log.info('Cleaning up...');
    
    if (this.pollTimeout) {
        clearTimeout(this.pollTimeout);
        this.pollTimeout = undefined;
    }
    if (this.connectionTimer) {
        clearTimeout(this.connectionTimer);
        this.connectionTimer = undefined;
    }
    // Close connections, clean up resources
    callback();
  } catch (e) {
    callback();
  }
}
```

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

## CI/CD and Testing Integration

### GitHub Actions for API Testing
For adapters with external API dependencies, implement separate CI/CD jobs:

```yaml
# Tests API connectivity with demo credentials (runs separately)
demo-api-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run demo API tests
      run: npm run test:integration-demo
```

### CI/CD Best Practices
- Run credential tests separately from main test suite
- Use ubuntu-22.04 for consistency
- Don't make credential tests required for deployment
- Provide clear failure messages for API connectivity issues
- Use appropriate timeouts for external API calls (120+ seconds)

### Package.json Script Integration
Add dedicated script for credential testing:
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

### Practical Example: Complete API Testing Implementation
Here's a complete example based on lessons learned from the Discovergy adapter:

#### test/integration-demo.js
```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Helper function to encrypt password using ioBroker's encryption method
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    
    if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
        throw new Error("Could not retrieve system secret for password encryption");
    }
    
    const secret = systemConfig.native.secret;
    let result = '';
    for (let i = 0; i < password.length; ++i) {
        result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
    }
    
    return result;
}

// Run integration tests with demo credentials
tests.integration(path.join(__dirname, ".."), {
    defineAdditionalTests({ suite }) {
        suite("API Testing with Demo Credentials", (getHarness) => {
            let harness;
            
            before(() => {
                harness = getHarness();
            });

            it("Should connect to API and initialize with demo credentials", async () => {
                console.log("Setting up demo credentials...");
                
                if (harness.isAdapterRunning()) {
                    await harness.stopAdapter();
                }
                
                const encryptedPassword = await encryptPassword(harness, "demo_password");
                
                await harness.changeAdapterConfig("your-adapter", {
                    native: {
                        username: "demo@provider.com",
                        password: encryptedPassword,
                        // other config options
                    }
                });

                console.log("Starting adapter with demo credentials...");
                await harness.startAdapter();
                
                // Wait for API calls and initialization
                await new Promise(resolve => setTimeout(resolve, 60000));
                
                const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
                
                if (connectionState && connectionState.val === true) {
                    console.log("✅ SUCCESS: API connection established");
                    return true;
                } else {
                    throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
                        "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
                }
            }).timeout(120000);
        });
    }
});
```

### Pollenflug-Specific Testing Patterns

For the pollenflug adapter, implement these specific test patterns:

```javascript
// Mock DWD API response for testing
const mockDWDResponse = {
    content: [
        {
            region_id: "31",
            region_name: "Westl. Niedersachsen/Bremen",
            Pollen: {
                Hasel: { today: "1", tomorrow: "2", dayafter_to: "1" },
                Erle: { today: "0", tomorrow: "1", dayafter_to: "2" },
                Birke: { today: "3", tomorrow: "2", dayafter_to: "1" }
                // ... other pollen types
            }
        }
    ]
};

// Test data processing
it('should process DWD pollen data correctly', async () => {
    const adapter = new PollenflugAdapter();
    await adapter.processPollenData(mockDWDResponse);
    
    // Verify states were created and updated
    const birchState = await adapter.getStateAsync('today.birch');
    expect(birchState.val).toBe(3);
});
```

### Logging and Debugging

#### Proper Logging Levels
```javascript
// Error - for errors that prevent functionality
this.log.error('Failed to fetch data from DWD API');

// Warn - for potentially problematic situations
this.log.warn('No specific region configured, using all regions');

// Info - for general information about adapter operation
this.log.info('Pollen data updated successfully');

// Debug - for detailed diagnostic information
this.log.debug('Processing region data for region ID: 31');
```

### Development Guidelines for Pollenflug Adapter

- Always validate DWD API responses before processing
- Handle network timeouts gracefully (DWD API can be slow)
- Support all German region codes as defined in io-package.json
- Maintain backward compatibility with existing state structure
- Update validity periods (info.today, info.tomorrow, info.dayaftertomorrow) correctly
- Handle the special case where DWD data may be from previous day until 11 AM update
- Implement proper error recovery for temporary API outages
- Use appropriate polling intervals (respect DWD server resources)