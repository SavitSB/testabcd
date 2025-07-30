import time

def queueRequests(target, wordlists):
    # pick Turbo Intruder’s HTTP/2 stack
    req_engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=10,
        pipeline=False,
        engine=engine.HTTP2
    )

    # throttle to ~10 000 RPM (≈167 RPS)
    interval = 60.0 / 10000
    end_time = time.time() + 60

    while time.time() < end_time:
        req_engine.queue(target.req)  # null payload
        time.sleep(interval)

    req_engine.start()

def handleResponse(req, interesting):
    table.add(req)
