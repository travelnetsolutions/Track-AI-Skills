# What Track asks for at certification

Sources:

- [Quick Start Guide](https://developer.trackhs.com/docs/quick-start-guide), section "Pre-Certification Considerations"
- [Introduction](https://developer.trackhs.com/docs/introduction): "If you are a vendor integrating your product into TRACK, we will work with you to certify your connection once it's completed."

## When

After development and testing of the integration is complete.

## What the developer sends

1. **A new, fresh set of test calls**, and the **time and date** they ran.
2. **Integration overview**: a brief summary of how the integration works
   and what key functionality it covers. Track uses it to build a testing
   plan around the use case.
3. **API call list**: the key API calls the integration uses, so Track can
   monitor them in the logs and validate their effects in the Track UI.
4. **Sandbox domain confirmation**: confirm the team has been testing
   against `https://api-integration-example.tracksandbox.io`.

## What Track does next

- Track's technical resource(s) review the activity in the API logs.
- If they find areas for improvement and cannot proceed, they provide
  guidance and ask for re-testing.
- If everything is satisfactory, they issue a **certificate of integration**,
  with additional resources for going live with mutual customers.

## Not published: do not claim

- Track's review criteria or a pass/fail checklist
- How long review takes, or any SLA
- Who the developer's Track technical contact is, or the address to send
  the package to (use the contact the developer already has; if they have
  none, the Quick Start points to the support form at
  https://support.trackhospitality.com/hc/en-us and Getting Help lists
  support@trackhs.com)
- Whether certification is required per property manager or once per
  integration
