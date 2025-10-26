Here is the XML for a professional architecture diagram that you can open directly in `draw.io`.

**Instructions:**

1.  Create a new, blank text file.
2.  Copy the entire XML code block below and paste it into your text file.
3.  Save the file with a `.drawio` or `.xml` extension (e.g., `HSBC_SDK_Architecture.drawio`).
4.  Open `draw.io` (or `app.diagrams.net`) and choose **File > Open from...** to select and open the file you just saved.

***

### Draw\.io XML Code

XML

```
<mxfile host="app.diagrams.net" modified="2025-10-26T15:30:00.000Z" agent="Gemini" etag="12345" version="22.0.3" type="device">
  <diagram name="Page-1" id="-S-123-abc">
    <mxGraphModel dx="1786" dy="1010" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1100" pageHeight="850" background="#ffffff" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
        <mxCell id="partner-app-container" value="Partner App (e.g., Ctrip)" style="swimlane;fontStyle=1;align=center;verticalAlign=top;childLayout=stackLayout;horizontal=1;startSize=30;horizontalStack=0;resizeParent=1;resizeParentMax=0;resizeLast=0;collapsible=1;marginBottom=0;html=1;strokeColor=#181818;fillColor=#dae8fc;fontColor=#181818;fontSize=14;" parent="1" vertex="1">
          <mxGeometry x="40" y="40" width="460" height="600" as="geometry" />
        </mxCell>
        <mxCell id="aosdk-container" value="AOSDK (HSBC) – iOS/Android" style="swimlane;fontStyle=1;align=center;verticalAlign=top;childLayout=stackLayout;horizontal=1;startSize=30;horizontalStack=0;resizeParent=1;resizeParentMax=0;resizeLast=0;collapsible=0;marginBottom=0;html=1;fillColor=#e1d5e7;strokeColor=#181818;fontColor=#181818;fontSize=12;" parent="partner-app-container" vertex="1">
          <mxGeometry x="20" y="50" width="420" height="420" as="geometry">
            <mxRectangle x="20" y="50" width="160" height="30" as="alternateBounds" />
          </mxGeometry>
        </mxCell>
        <mxCell id="component-core" value="Core (Session, Tokens, Config, Logger, Events)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="aosdk-container" vertex="1">
          <mxGeometry y="30" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="component-security" value="Security (OIDC+PKCE, Key Pinning, Attestation Reporter)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="aosdk-container" vertex="1">
          <mxGeometry y="80" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="component-ui" value="UI Container (WebView / Custom Tab / ASWebAuthenticationSession)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="aosdk-container" vertex="1">
          <mxGeometry y="130" width="420" height="60" as="geometry" />
        </mxCell>
        <mxCell id="component-kyc-loader" value="On-Demand KYC Loader (loads modules)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="aosdk-container" vertex="1">
          <mxGeometry y="190" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="component-prefill" value="Prefill Adapter (Token-based prefill)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="aosdk-container" vertex="1">
          <mxGeometry y="240" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="component-appointment" value="Appointment &amp;amp; Status Adapters" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="aosdk-container" vertex="1">
          <mxGeometry y="290" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="component-kyc-modules" value="[Optional Native KYC Modules]&lt;br&gt;(Downloaded On-Demand)" style="shape=stack;overflow=hidden;shadow=0;dashed=1;align=center;verticalAlign=middle;strokeColor=#181818;fillColor=#f5f5f5;fontStyle=2;fontSize=11;" parent="partner-app-container" vertex="1">
          <mxGeometry x="20" y="490" width="420" height="70" as="geometry" />
        </mxCell>
        <mxCell id="hsbc-gateway-container" value="HSBC Partner Gateway" style="swimlane;fontStyle=1;align=center;verticalAlign=top;childLayout=stackLayout;horizontal=1;startSize=30;horizontalStack=0;resizeParent=1;resizeParentMax=0;resizeLast=0;collapsible=1;marginBottom=0;html=1;strokeColor=#181818;fillColor=#d5e8d4;fontColor=#181818;fontSize=14;" parent="1" vertex="1">
          <mxGeometry x="640" y="40" width="420" height="600" as="geometry" />
        </mxCell>
        <mxCell id="service-oauth" value="OAuth/OIDC (Authz Code + PKCE)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="30" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="service-prefill" value="Prefill API (server-to-server)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="80" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="service-ao-web" value="AO Web (responsive pages + web-based KYC fallbacks)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="130" width="420" height="60" as="geometry" />
        </mxCell>
        <mxCell id="service-risk" value="Risk &amp;amp; Policy Engine (consumes attestation data)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="190" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="service-kyc-aml" value="KYC/AML/Screening (internal or 3rd-party)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="240" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="service-doc" value="Document Service (upload, OCR, classification)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="290" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="service-orchestrator" value="Application Orchestrator (BPMN/Rules)" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="340" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="service-status" value="Application Status API" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=middle;whiteSpace=wrap;rounded=0;spacingLeft=10;" parent="hsbc-gateway-container" vertex="1">
          <mxGeometry y="390" width="420" height="50" as="geometry" />
        </mxCell>
        <mxCell id="connector-arrow" value="Internet&lt;br&gt;(TLS + Public Key Pinning, JWS/JWE)" style="endArrow=classic;html=1;rounded=0;exitX=1;exitY=0.5;entryX=0;entryY=0.5;strokeColor=#181818;strokeWidth=2;" parent="1" source="aosdk-container" target="hsbc-gateway-container" edge="1">
          <mxGeometry width="50" height="50" relative="1" as="geometry">
            <mxPoint x="500" y="340" as="sourcePoint" />
            <mxPoint x="550" y="290" as="targetPoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="kyc-loader-arrow" value="" style="endArrow=classic;html=1;rounded=0;entryX=0.5;entryY=0;exitX=0.5;exitY=1;strokeColor=#181818;dashed=1;" parent="1" source="component-kyc-loader" target="component-kyc-modules" edge="1">
          <mxGeometry width="50" height="50" relative="1" as="geometry">
            <mxPoint x="270" y="480" as="sourcePoint" />
            <mxPoint x="320" y="430" as="targetPoint" />
          </mxGeometry>
        </mxCell>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>

```

