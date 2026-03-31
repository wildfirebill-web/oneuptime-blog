# How to Use Dapr for Healthcare System Integration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Healthcare, HIPAA, Integration, Microservice

Description: Build HIPAA-compliant healthcare system integrations using Dapr for EHR connectivity, patient data management, and clinical workflow orchestration.

---

Healthcare system integration involves connecting EHRs (Electronic Health Records), lab systems, pharmacy platforms, and clinical workflows while maintaining HIPAA compliance. Dapr's security features (mTLS, cryptography, secret management) and integration patterns (pub/sub for HL7/FHIR events, workflows for clinical pathways) make it a strong foundation for healthcare microservices.

## Healthcare Architecture with Dapr

```text
EHR System -> HL7/FHIR Adapter Service -> Dapr Pub/Sub
                                        -> Patient Service
                                        -> Lab Service
                                        -> Pharmacy Service
                                        -> Notification Service
```

Each service uses Dapr for encrypted communication and audit logging, with all PHI (Protected Health Information) encrypted at rest.

## Patient Data Service with Encryption

Encrypt all PHI before storing:

```python
# patient_service/patient.py
from dapr.clients import DaprClient
import json
import base64

CRYPTO_COMPONENT = "azure-keyvault"
PHI_KEY = "patient-data-key"
STATE_STORE = "patient-statestore"

# PHI fields requiring encryption
PHI_FIELDS = {'ssn', 'date_of_birth', 'diagnosis', 'medications',
              'insurance_id', 'address', 'phone'}

def save_patient(patient_id: str, patient_data: dict):
    with DaprClient() as client:
        encrypted_data = {}

        for field, value in patient_data.items():
            if field in PHI_FIELDS and value:
                plaintext = str(value).encode('utf-8')
                encrypted = client.encrypt(
                    data=plaintext,
                    options={"componentName": CRYPTO_COMPONENT,
                             "keyName": PHI_KEY, "algorithm": "A256GCM"}
                )
                encrypted_data[field] = base64.b64encode(encrypted.data).decode()
                encrypted_data[f"_phi_{field}"] = True
            else:
                encrypted_data[field] = value

        client.save_state(STATE_STORE, f"patient:{patient_id}",
                          json.dumps(encrypted_data))

        # Publish de-identified event (no PHI in events)
        client.publish_event("healthcare-pubsub", "patient-updated", {
            "patient_id": patient_id,
            "updated_fields": [f for f in patient_data if f not in PHI_FIELDS],
            "timestamp": "now"
        })
```

## FHIR Resource Integration

Handle FHIR resources via Dapr bindings:

```python
# fhir_adapter/adapter.py
from flask import Flask, request, jsonify
from dapr.clients import DaprClient
import json

app = Flask(__name__)

@app.route('/fhir/Patient', methods=['POST'])
def create_fhir_patient():
    """Receives FHIR Patient resource and routes to patient service"""
    fhir_patient = request.json

    # Transform FHIR to internal format
    internal_patient = transform_fhir_to_internal(fhir_patient)

    with DaprClient() as client:
        result = client.invoke_method(
            app_id="patient-service",
            method_name="patients",
            http_verb="POST",
            data=json.dumps(internal_patient).encode()
        )
    return jsonify(json.loads(result.data)), 201

def transform_fhir_to_internal(fhir_patient: dict) -> dict:
    name = fhir_patient.get('name', [{}])[0]
    return {
        'fhir_id': fhir_patient.get('id'),
        'first_name': name.get('given', [''])[0],
        'last_name': name.get('family', ''),
        'date_of_birth': fhir_patient.get('birthDate'),
        'gender': fhir_patient.get('gender'),
        'identifier': next(
            (i['value'] for i in fhir_patient.get('identifier', [])
             if i.get('system', '').endswith('ssn')), None
        )
    }
```

## Clinical Workflow Orchestration

Manage lab order workflows with Dapr Workflow:

```python
# lab_service/workflows/lab_order_workflow.py
import dapr.ext.workflow as wf

@wf.workflow
def lab_order_workflow(ctx, order: dict):
    # Step 1: Validate order
    validation = yield ctx.call_activity(validate_lab_order, input=order)
    if not validation['approved']:
        return {'status': 'rejected', 'reason': validation['reason']}

    # Step 2: Route to appropriate lab
    lab_assignment = yield ctx.call_activity(assign_lab, input=order)

    # Step 3: Wait for results (may take hours)
    results = yield ctx.wait_for_external_event("lab-results-received",
                                                 timeout_in_seconds=86400)

    # Step 4: Notify clinician
    yield ctx.call_activity(notify_clinician, input={
        'order': order,
        'results': results,
        'lab': lab_assignment
    })

    # Step 5: Update EHR
    yield ctx.call_activity(update_ehr, input={
        'patient_id': order['patient_id'],
        'results': results
    })

    return {'status': 'completed', 'order_id': order['id']}
```

## HIPAA Audit Logging

Every data access must be logged for HIPAA compliance:

```python
# middleware/hipaa_audit.py
from functools import wraps
from flask import request, g
from dapr.clients import DaprClient
import json

def hipaa_audit_log(resource_type: str, action: str):
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            result = f(*args, **kwargs)

            audit_entry = {
                'user_id': request.headers.get('X-User-Id', 'unknown'),
                'resource_type': resource_type,
                'action': action,
                'resource_id': kwargs.get('patient_id', 'unknown'),
                'timestamp': __import__('datetime').datetime.utcnow().isoformat(),
                'ip_address': request.remote_addr,
                'status': 'success'
            }

            with DaprClient() as client:
                client.publish_event("healthcare-pubsub", "phi-access", audit_entry)

            return result
        return decorated
    return decorator

@app.route('/patients/<patient_id>')
@hipaa_audit_log('Patient', 'read')
def get_patient(patient_id: str):
    # ... implementation
    pass
```

## Summary

Dapr enables HIPAA-compliant healthcare integrations through field-level PHI encryption using the Cryptography API, mTLS for encrypted service communication, Workflow for multi-step clinical processes, and pub/sub for comprehensive audit trails. The de-identification of event payloads ensures PHI stays protected even in messaging infrastructure.
