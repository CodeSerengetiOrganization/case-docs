# The equipment master tracks the rack, the fixture, and the nests

The plant must know which rack, which fixture, and which nest held the DUT. That record is the MES equipment master.

The CASE contract stores the ids only. The CASE contract does not hold these tables. Contract: [02-contract-store-the-spike.md](../../scored-eol-cause-list/design/02-contract-store-the-spike.md).

This issue is not part of "Scored EOL cause list." It does not change that score.

Close this issue when the equipment master tracks the rack, the fixture, and the nests, and the CASE contract still does not hold those tables.

## Reference

The machine-monitoring notes already sketch this. `equipment_hierarchy` holds the rack as the parent and the fixture as the child. A nest is a child of the fixture.

The notes use the word controller for the rack. The CASE glossary uses rack.

Source: `Note-Machine-Monitoring-System-Epic1/MMS-DB/1-new-table-design-for-manufacturing-result.md`

The manufacturing result keeps the station setup as columns named with the CASE glossary: `station_id`, `rack_id`, `fixture_id`, and `nest_number`. MES does not use `controller_id`. That decision is temporary and moves to the MES project. Note: [02-temporary-note-manufacturing-result-keeps-the-equipment-ids.md](../../../notes/02-temporary-note-manufacturing-result-keeps-the-equipment-ids.md).
