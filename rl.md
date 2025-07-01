# MDT Receiver Rate Limiting Tests

This document outlines and validates rate limiting behavior in `mdtreceiver` under a variety of configurations and edge cases.

## Table of Contents
- [Test Case: General Wildcard Match](#tc_mdt_receiver_config_mostspecificmatch_general)
- [Test Case: Specific Rule Takes Priority](#tc_mdt_receiver_config_mostspecificmatch_specific)
- [Test Case: All Channel Space Matches](#tc_mdt_receiver_config_allmatches)
- [Test Case: Pre-Upload Rejected (429)](#tc_mdt_receiver_preupload_rejected_429)
- [Test Case: Post-Upload File Limit (429)](#tc_mdt_receiver_postupload_files_rejected_429)
- [Test Case: System Limit (503)](#tc_mdt_receiver_systemlimit_exceeded_503)
- [Test Case: Global Warn Mode Allowed](#tc_mdt_receiver_globalwarnmode_true)
- [Test Case: Warn Mode Overridden by Channel Config](#tc_mdt_receiver_specificwarnmode_override)
- [Test Case: Bouncer Error—Fail Open](#tc_mdt_receiver_bouncererror_failopen)
- [Test Case: Metrics Logging Consistency](#tc_mdt_receiver_metrics_logging_consistency)
