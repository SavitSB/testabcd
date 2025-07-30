from turbo_intruder import RequestEngine, engine
import time

def queueRequests(target, wordlists):
    # Use Turbo Intruder’s custom HTTP/2 engine
    req_engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=10,
        pipeline=False,
        engine=engine.HTTP2      # ← HTTP/2 via Turbo Intruder’s built‑in stack
    )

    # Calculate sleep interval for ~10 000 RPM
    interval = 60.0 / 10000     # ≈0.006 s per request
    end_time = time.time() + 60 # run for one minute

    while time.time() < end_time:
        req_engine.queue(target.req)  # queue the null (unmodified) payload
        time.sleep(interval)          # throttle

    req_engine.start()               # send all queued requests

def handleResponse(req, interesting):
    table.add(req)
