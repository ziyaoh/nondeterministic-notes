# Design Data Intensive Application

## Foundation of Data Systems

### Reliable, Scalable, and Maintainable Applications

#### Reliability: fault-tolerant / resilient

- Hardware Faults
- Software Errors
- Human Errors

#### Scalability

Scalability depends on the load parameter we care about: if the system grows in **a particular way**, what are the options for coping with the growth? Meaning we have to consider the specific use case.

load parameters
- requests per second for web servers
- ratio of reads to wries for databases
- hit rate for a caches
- etc.

**Performance**
- throughput for batch job
- response time for online system

    use percentile for better understanding of the service performance

**Scaling**
- scale up
- scal out

#### Maintainability
- operability: make life easy for operations
- simplicity: clean code basically
- evlvability: make changes easy