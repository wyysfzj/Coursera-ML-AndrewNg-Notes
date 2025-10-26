Here is the XML for a professional sequence diagram based on Section 18, which you can open directly in `draw.io`.

**Instructions:**

1.  Create a new, blank text file.
2.  Copy the entire XML code block below and paste it into your text file.
3.  Save the file with a `.drawio` or `.xml` extension (e.g., `HSBC_AO_Sequence.drawio`).
4.  Open `draw.io` (or `app.diagrams.net`) and choose **File > Open from...** to select and open the file you just saved.

***

### Draw\.io XML Code

XML

```
<mxfile host="app.diagrams.net" modified="2025-10-26T15:45:00.000Z" agent="Gemini" etag="67890" version="22.0.3" type="device">
  <diagram name="Page-1" id="abc-123-xyz">
    <mxGraphModel dx="1422" dy="1733" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1169" pageHeight="827" background="#ffffff" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
        <mxCell id="actor-user" value="Customer" style="shape=umlActor;verticalLabelPosition=bottom;verticalAlign=top;html=1;outlineConnect=0;fontColor=#181818;strokeColor=#181818;" parent="1" vertex="1">
          <mxGeometry x="60" y="80" width="30" height="60" as="geometry" />
        </mxCell>
        <mxCell id="lifeline-ctrip-app" value="Ctrip App (AOSDK)" style="shape=umlLifeline;perimeter=lifelinePerimeter;whiteSpace=wrap;html=1;container=1;dropTarget=0;collapsible=0;recursiveResize=0;outlineConnect=0;portConstraint=eastwest;newEdgeStyle={&quot;edgeStyle&quot;:&quot;elbowEdgeStyle&quot;,&quot;elbow&quot;:&quot;vertical&quot;,&quot;curved&quot;:0,&quot;rounded&quot;:0};fontColor=#181818;strokeColor=#181818;fillColor=#dae8fc;" parent="1" vertex="1">
          <mxGeometry x="160" y="80" width="180" height="1000" as="geometry" />
        </mxCell>
        <mxCell id="lifeline-ctrip-backend" value="Ctrip Backend" style="shape=umlLifeline;perimeter=lifelinePerimeter;whiteSpace=wrap;html=1;container=1;dropTarget=0;collapsible=0;recursiveResize=0;outlineConnect=0;portConstraint=eastwest;newEdgeStyle={&quot;edgeStyle&quot;:&quot;elbowEdgeStyle&quot;,&quot;elbow&quot;:&quot;vertical&quot;,&quot;curved&quot;:0,&quot;rounded&quot;:0};fontColor=#181818;strokeColor=#181818;fillColor=#f8cecc;" parent="1" vertex="1">
          <mxGeometry x="400" y="80" width="180" height="1000" as="geometry" />
        </mxCell>
        <mxCell id="lifeline-hsbc-gateway" value="HSBC Partner GW" style="shape=umlLifeline;perimeter=lifelinePerimeter;whiteSpace=wrap;html=1;container=1;dropTarget=0;collapsible=0;recursiveResize=0;outlineConnect=0;portConstraint=eastwest;newEdgeStyle={&quot;edgeStyle&quot;:&quot;elbowEdgeStyle&quot;,&quot;elbow&quot;:&quot;vertical&quot;,&quot;curved&quot;:0,&quot;rounded&quot;:0};fontColor=#181818;strokeColor=#181818;fillColor=#d5e8d4;" parent="1" vertex="1">
          <mxGeometry x="640" y="80" width="180" height="1000" as="geometry" />
        </mxCell>
        <mxCell id="lifeline-hsbc-services" value="HSBC AO Web &amp;amp; Services" style="shape=umlLifeline;perimeter=lifelinePerimeter;whiteSpace=wrap;html=1;container=1;dropTarget=0;collapsible=0;recursiveResize=0;outlineConnect=0;portConstraint=eastwest;newEdgeStyle={&quot;edgeStyle&quot;:&quot;elbowEdgeStyle&quot;,&quot;elbow&quot;:&quot;vertical&quot;,&quot;curved&quot;:0,&quot;rounded&quot;:0};fontColor=#181818;strokeColor=#181818;fillColor=#d5e8d4;" parent="1" vertex="1">
          <mxGeometry x="880" y="80" width="220" height="1000" as="geometry" />
        </mxCell>
        <mxCell id="msg-1" value="1. Tap &quot;Open HSBC Account&quot;" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=20;strokeColor=#181818;fontColor=#181818;" parent="1" source="actor-user" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="120" y="140" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-2" value="2. Request Prefill Token (with consent)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=60;strokeColor=#181818;fontColor=#181818;" parent="1" source="lifeline-ctrip-app" target="lifeline-ctrip-backend" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-3" value="3. POST /prefill (PII, consent, mTLS, JWE)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=80;strokeColor=#181818;fontColor=#181818;" parent="1" source="lifeline-ctrip-backend" target="lifeline-hsbc-gateway" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-4" value="4. prefillToken (JWS)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=120;strokeColor=#181818;fontColor=#181818;dashed=1;" parent="1" source="lifeline-hsbc-gateway" target="lifeline-ctrip-backend" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-5" value="5. Return prefillToken" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=160;strokeColor=#181818;fontColor=#181818;dashed=1;" parent="1" source="lifeline-ctrip-backend" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-6" value="6. HSBCAOSdk.start(config, params)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=200;strokeColor=#181818;fontColor=#181818;" parent="1" source="lifeline-ctrip-app" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="frame-security" value="SDK Security Handshake" style="shape=umlFrame;strokeColor=#181818;fontColor=#181818;fillColor=none;" parent="1" vertex="1">
          <mxGeometry x="140" y="320" width="700" height="200" as="geometry" />
        </mxCell>
        <mxCell id="msg-7" value="7. Version Check( )" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=20;strokeColor=#181818;fontColor=#181818;" parent="frame-security" source="lifeline-ctrip-app" target="lifeline-hsbc-gateway" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="250" y="60" as="targetPoint" />
            <mxPoint x="90" y="60" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-8" value="8. { minRequiredVersion: '...' }" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=40;strokeColor=#181818;fontColor=#181818;dashed=1;" parent="frame-security" source="lifeline-hsbc-gateway" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="90" y="80" as="targetPoint" />
            <mxPoint x="250" y="80" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-9" value="9. OIDC Auth Request (w/ Attestation Token)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=80;strokeColor=#181818;fontColor=#181818;" parent="frame-security" source="lifeline-ctrip-app" target="lifeline-hsbc-gateway" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="250" y="120" as="targetPoint" />
            <mxPoint x="90" y="120" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-10" value="10. Server Policy Check (Version &amp;amp; Attestation OK)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=100;strokeColor=#181818;fontColor=#181818;" parent="frame-security" source="lifeline-hsbc-gateway" target="lifeline-hsbc-gateway" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="280" y="140" as="targetPoint" />
            <mxPoint x="280" y="140" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-11" value="11. Auth Code" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=140;strokeColor=#181818;fontColor=#181818;dashed=1;" parent="frame-security" source="lifeline-hsbc-gateway" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="90" y="180" as="targetPoint" />
            <mxPoint x="250" y="180" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-12" value="12. Open AO Web (w/ Auth Code &amp;amp; Prefill Token)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=480;strokeColor=#181818;fontColor=#181818;" parent="1" source="lifeline-ctrip-app" target="lifeline-hsbc-services" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-13" value="13. Redeem Prefill Token" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=520;strokeColor=#181818;fontColor=#181818;" parent="1" source="lifeline-hsbc-services" target="lifeline-hsbc-gateway" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-14" value="14. Prefilled Data" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=560;strokeColor=#181818;fontColor=#181818;dashed=1;" parent="1" source="lifeline-hsbc-gateway" target="lifeline-hsbc-services" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="frame-ao" value="alt" style="shape=umlFrame;strokeColor=#181818;fontColor=#181818;fillColor=none;" parent="1" vertex="1">
          <mxGeometry x="140" y="660" width="980" height="300" as="geometry" />
        </mxCell>
        <mxCell id="frame-kyc-native" value="[ Native KYC ]" style="shape=umlFrame;strokeColor=#181818;fontColor=#181818;fillColor=none;" parent="frame-ao" vertex="1">
          <mxGeometry y="30" width="980" height="120" as="geometry" />
        </mxCell>
        <mxCell id="msg-15-native" value="15a. Load On-Demand KYC Module" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=20;strokeColor=#181818;fontColor=#181818;" parent="frame-kyc-native" source="lifeline-ctrip-app" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="50" y="60" as="targetPoint" />
            <mxPoint x="50" y="60" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-16-native" value="16a. Request Native KYC (e.g., Doc Scan)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=40;strokeColor=#181818;fontColor=#181818;" parent="frame-kyc-native" source="lifeline-hsbc-services" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="90" y="80" as="targetPoint" />
            <mxPoint x="770" y="80" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-17-native" value="17a. Return Encrypted KYC Data" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=80;strokeColor=#181818;fontColor=#181818;dashed=1;" parent="frame-kyc-native" source="lifeline-ctrip-app" target="lifeline-hsbc-services" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="770" y="120" as="targetPoint" />
            <mxPoint x="90" y="120" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="frame-kyc-web" value="[ Web KYC Fallback ]" style="shape=umlFrame;strokeColor=#181818;fontColor=#181818;fillColor=none;" parent="frame-ao" vertex="1">
          <mxGeometry y="150" width="980" height="150" as="geometry" />
        </mxCell>
        <mxCell id="msg-15-web" value="15b. (Native KYC fails or is disabled)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=20;strokeColor=#181818;fontColor=#181818;fontStyle=2" parent="frame-kyc-web" source="lifeline-ctrip-app" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="50" y="60" as="targetPoint" />
            <mxPoint x="50" y="60" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="msg-16-web" value="16b. User completes KYC in WebView" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=60;strokeColor=#181818;fontColor=#181818;" parent="frame-kyc-web" source="lifeline-hsbc-services" target="lifeline-hsbc-services" edge="1">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="800" y="100" as="targetPoint" />
            <mxPoint x="800" y="100" as="sourcePoint" />
          </mxGeometry>
        </mxCell>
        <mxCell id="separator" value="" style="line;strokeWidth=1;fillColor=none;strokeColor=#181818;dashed=1;" parent="frame-ao" vertex="1">
          <mxGeometry y="150" width="980" height="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-18" value="18. Submit Application" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=900;strokeColor=#181818;fontColor=#181818;" parent="1" source="lifeline-hsbc-services" target="lifeline-hsbc-services" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-19" value="19. applicationId &amp; state" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=940;strokeColor=#181818;fontColor=#181818;dashed=1;" parent="1" source="lifeline-hsbc-services" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
        <mxCell id="msg-20" value="20. AOResult.Completed(applicationId)" style="edgeStyle=orthogonalEdgeStyle;rounded=0;outlineConnect=0;targetFinder=;entryX=0;entryY=960;strokeColor=#181818;fontColor=#181818;" parent="1" source="lifeline-ctrip-app" target="lifeline-ctrip-app" edge="1">
          <mxGeometry relative="1" as="geometry" />
        </mxCell>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>

```

