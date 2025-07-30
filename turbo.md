import time

def queueRequests(target, wordlists):
    # Configure the engine for high throughput
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=10,
        pipeline=False
    )

    # Calculate sleep interval to hit ~10000 RPM
    interval = 60.0 / 10000  # ~0.006 seconds per request

    # Send requests for 60 seconds at the calculated rate
    start_time = time.time()
    while time.time() - start_time < 60:
        engine.queue(target.req)    # queue the unmodified (null) payload
        time.sleep(interval)        # throttle to maintain target RPM

    # Kick off the attack
    engine.start()

def handleResponse(req, interesting):
    # Log all responses to the results table
    table.add(req)
