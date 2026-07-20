# security footage assessment

- [scope](#scope)
- [reconnaissance and evidence acquisition](#reconnaissance-and-evidence-acquisition)
- [security findings and validation](#security-findings-and-validation)
- [impact and remediation](#impact-and-remediation)
- [lessons learned](#lessons-learned)
- [analyst notes](#analyst-notes)
- [references](#references)

## scope

### authorization and objective

This assessment documents forensic analysis performed against the downloadable task file provided by the authorized TryHackMe Security Footage room. The objective was to recover security-camera footage from a network capture and validate the recovered evidence without modifying the source capture.

### room information

- Type: challenge
- Difficulty: medium
- Focus: digital forensics and network capture analysis
- Estimated duration: approximately 45 minutes

The room scenario describes an office break-in where the original security-footage storage was destroyed. The supplied network capture is therefore treated as the primary evidence source.

Room link: [TryHackMe Security Footage](https://tryhackme.com/room/securityfootage)

### target and testing environment

- Evidence source: downloadable PCAP/task files
- Analysis environment: Kali Linux terminal, Wireshark, and a local extraction script
- Network endpoint visible in the capture: `192.168.1.100:8081`
- No target VM or live service was required for the documented activity

The screenshots are room evidence. The recovered flag value is intentionally not reproduced in the narrative, commands, or code blocks.

## reconnaissance and evidence acquisition

### packet capture and HTTP stream inspection

The capture contained an HTTP request to `192.168.1.100:8081`. The response used `multipart/x-mixed-replace` with a boundary and JPEG parts, indicating a multipart image stream rather than a conventional single web page.

![HTTP multipart response carrying JPEG data](Images/security_footage/Screenshot%20From%202026-07-20%2017-37-39.png)

*Figure 1 — A TCP stream contains an HTTP response with multipart JPEG content and visible JPEG file markers.*

The unencrypted HTTP stream made the image data available for reconstruction from the capture. The packet data shown is sufficient to justify extracting the embedded image parts.

### image extraction workflow

The capture data was staged locally and processed with an extraction script. The command shown below separates embedded image data into an output directory:

```text
python3 extract_images.py image/image.bin extracted_images
```

![Image extraction command and staged capture files](Images/security_footage/Screenshot%20From%202026-07-20%2017-36-36.png)

*Figure 2 — The capture-derived image data is processed into individual extracted images.*

The first extraction output lists byte ranges for successive JPEG objects. This provides a reproducible relationship between the capture data and the recovered artifacts.

### extraction result validation

The extraction completed with 541 images. The output directory was then opened for visual inspection.

![Completed extraction of 541 image frames](Images/security_footage/Screenshot%20From%202026-07-20%2017-36-25.png)

*Figure 3 — The extraction process reports 541 recovered images before visual review.*

### recovered footage review

The extracted JPEG set showed a sequence of frames containing a sign whose text becomes progressively recoverable across the image set.

![Recovered image frames showing the reconstructed security-footage flag](Images/security_footage/Screenshot%20From%202026-07-20%2017-36-13.png)

*Figure 4 — The recovered frame set provides the visual evidence needed to reconstruct the room answer.*

The visual result confirms that the footage was successfully recovered from the network capture. The exact answer is omitted from this report to keep challenge secrets out of the written narrative.

## security findings and validation

### finding 1: security footage transmitted over unencrypted HTTP

**Status:** confirmed in the supplied capture

The captured traffic uses HTTP rather than HTTPS and carries JPEG frames inside a multipart response. Anyone able to obtain equivalent network visibility could reconstruct the footage from the stream.

This is a transport-security observation demonstrated by the lab capture. The screenshots do not establish whether the same configuration exists outside the authorized challenge environment.

### finding 2: recoverable surveillance frames in retained network traffic

**Status:** confirmed in the supplied capture

The PCAP retained enough application-layer data to reconstruct 541 individual JPEG images. The extracted frames reveal the camera content and allow the security-footage message to be recovered even though the original storage was reportedly destroyed.

This demonstrates that network captures containing surveillance streams are sensitive evidence and must be protected like the original footage.

### evidence validation status

The current evidence supports the following conclusions:

- the capture contains an HTTP multipart image stream;
- the stream contains JPEG data;
- the data can be separated into 541 image files; and
- visual review of the extracted frames recovers the room’s hidden message.

The supplied screenshots do not provide a cryptographic hash, packet count, full capture timeline, or formal chain-of-custody record. Those items remain useful if the analysis is being documented as a real forensic investigation rather than a training-room exercise.

### pending validation

The following activities remain optional follow-up work:

- record SHA-256 hashes for the original PCAP and extracted artifacts;
- document the capture’s packet count, protocol hierarchy, and complete endpoint set;
- preserve the original capture as read-only evidence and perform analysis on a working copy;
- verify that each extracted object is a valid JPEG and preserve extraction logs; and
- document frame ordering and timestamps if the source stream exposes them.

These are evidence-quality improvements, not additional confirmed findings.

## impact and remediation

### impact

If this traffic pattern existed on a production network, unencrypted surveillance frames could be observed and reconstructed by any party with access to the relevant network segment or capture files. The recovered content could expose people, locations, and security-sensitive events.

### remediation

- Serve camera streams over TLS with authenticated clients and strong certificate validation.
- Isolate camera and recording systems on a protected network segment and restrict access to authorized monitoring systems.
- Avoid capturing or retaining raw surveillance traffic unless operationally necessary; apply access controls and defined retention periods to PCAP files.
- Encrypt stored footage and forensic exports, and maintain integrity hashes for evidence copies.
- Record capture provenance, timestamps, extraction tooling, and analyst actions when footage may support an investigation.

## lessons learned

- A network capture can retain recoverable application data even when the original storage is unavailable.
- HTTP content type, multipart boundaries, and JPEG markers are useful indicators during stream analysis.
- Extraction output should be validated by both object counts and visual review.
- Tool warnings should be separated from evidence about the capture itself.
- Sensitive video and PCAP files require comparable confidentiality and integrity controls.

## references

- [TryHackMe Security Footage room](https://tryhackme.com/room/securityfootage)
- [Wireshark User's Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [RFC 2046: Multipurpose Internet Mail Extensions](https://www.rfc-editor.org/rfc/rfc2046)
