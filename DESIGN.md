# Goals

We want to build a distributed computer to computer data pipeline. This will allow for such use cases as talking to a remote termial from a local machine for checking on machine health, installing software, etc.

# Desgin

We want to make this data pipeline be peer to peer over the internet. The connection will be initiated via web rtc. Note there is no "web" involved here, we just want to use the existing STUN/TURN software to set up the connections.

# Reqs

We do have access to a STUN/TURN server running on aws, so we can use that to facilitate connections.

For authentication, we would like to use OAUTH in some way to authenticate the connections.

# UI

The user on a terminal can run our command line, authenticate, and then get a list of remote terminal sessions it could connect to, and then allow the user to select one and connect to it. This will require that the server side of our software was already set up and running on the remote computer so it can register with the STUN server and have an open terminal for talking.

# Lanugage

Whatever has tbe best RTC support. Ideally something in the golang, python, c++ world. I believe that golang does have good support?
