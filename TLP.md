Basic description template for TLP that can be used elsewhere and used to tag alerts/incidents by playbooks


![2023-08-13 14_26_30-](https://github.com/Truvis/Sentinel/assets/23244379/12d4c9b1-8b7e-4956-98e7-b0e70ef30cff)

```
<div style="background-color: #000;border-left: 3px solid #f44336;"><p style="padding: 5px">🚦 <strong style="color:#FF2B2B">TLP:RED - Not for disclosure, restricted to participants only</strong></p></div><p style="padding:3px;border:1px solid #f44336;background-color:#fefefe">Sources may use TLP:RED when information cannot be effectively acted upon without significant risk for the privacy, reputation, or operations of the organizations involved. For the eyes and ears of individual recipients only, no further.
<br>
Recipients may not share TLP:RED information with any parties outside of the specific exchange, meeting, or conversation in which it was originally disclosed. In the context of a meeting, for example, TLP:RED information is limited to those present at the meeting. In most circumstances, TLP:RED should be exchanged verbally or in person.</p>

<div style="background-color: #000;border-left: 3px solid #FFC000;"><p style="padding: 5px">🚦 <strong style="color:#FFC000">TLP:AMBER+STRCIT - Limited disclosure, restricted to participants’ organization</strong></p></div><p style="padding:3px;border:1px solid #FFC000;background-color:#fefefe">Sources may use TLP:AMBER+STRICT when information requires support to be effectively acted upon, yet carries risk to privacy, reputation, or operations if shared outside of the organization.
<br>
Recipients may share TLP:AMBER+STRICT information only with members of their own organization on a need-to-know basis to protect their organization and prevent further harm.
</p>

<div style="background-color: #000;border-left: 3px solid #FFC000;"><p style="padding: 5px">🚦 <strong style="color:#FFC000">TLP:AMBER - Limited disclosure, restricted to participants’ organization and its clients (see Terminology Definitions).</strong></p></div><p style="padding:3px;border:1px solid #FFC000;background-color:#fefefe">Sources may use TLP:AMBER when information requires support to be effectively acted upon, yet carries risk to privacy, reputation, or operations if shared outside of the organizations involved. Note that TLP:AMBER+STRICT should be used to restrict sharing to the recipient organization only.
<br>
Recipients may share TLP:AMBER information with members of their own organization and its clients on a need-to-know basis to protect their organization and its clients and prevent further harm.
</p>

<div style="background-color: #000;border-left: 3px solid #33FF00;"><p style="padding: 5px">🚦 <strong style="color:#33FF00">TLP:GREEN - Limited disclosure, restricted to the community.</strong></p></div><p style="padding:3px;border:1px solid #33FF00;background-color:#fefefe">Sources may use TLP:GREEN when information is useful to increase awareness within their wider community.
<br>
Recipients may share TLP:GREEN information with peers and partner organizations within their community, but not via publicly accessible channels. Unless otherwise specified, TLP:GREEN information may not be shared outside of the cybersecurity or cyber defense community.
</p>

<div style="background-color: #000;border-left: 3px solid #FFFFFF;"><p style="padding: 5px">🚦 <strong style="color:#FFFFFF">TLP:CLEAR - Disclosure is not limited.</strong></p></div><p style="padding:3px;border:1px solid #FFFFFF;background-color:#fefefe">
Sources may use TLP:CLEAR when information carries minimal or no foreseeable risk of misuse, in accordance with applicable rules and procedures for public release.
<br>
Recipients may share this information without restriction. Information is subject to standard copyright rules.
</p>
```
