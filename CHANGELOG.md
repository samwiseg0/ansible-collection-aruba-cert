# Changelog

## 1.0.1 - 2026-09-29

- The serial number read tries each node of the path in order. It stops only when no node answers.
- Output without a serial number counts as a failed read.
- After the register step, the member check compares the serial number with the store certificate.
- The scp password check ignores a trailing CR or LF.
- The report reads what each node serves and names the nodes that an apply run would restart.
- The report probe tries 3 times, 5 seconds apart, before it counts a node as stuck.
- The served certificate waits poll inside the script for 60 seconds. ansible-core 2.19 no longer
  prints a false error.
- The report string has no semicolon, and the docs cover a node that restarts on every apply run.
- ssh runs with LogLevel=ERROR. The first-contact warning no longer breaks the serial number read.

## 1.0.0 - 2026-09-29

The first release adds the `server_cert` role.
