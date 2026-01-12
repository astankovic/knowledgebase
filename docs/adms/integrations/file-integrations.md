---
layout: default
title: File Integrations
parent: Integrations
grand_parent: ADMS Overview
nav_order: 3
---

# File Integrations

## Overview

File-based integrations provide bulk data exchange between ADMS and external systems using structured file formats. These integrations are appropriate for periodic data synchronization that doesn't require real-time updates: daily network model imports from GIS, hourly load forecasts from energy management systems, weekly maintenance schedules from work management systems, and monthly customer billing data from customer information systems.

File integrations offer several advantages including simplicity of implementation, compatibility with diverse systems, built-in batch processing characteristics, and ease of troubleshooting through direct file inspection. However, they introduce latency between data creation and consumption, require robust error handling for malformed files, and need coordination of file exchange timing between systems.

The file integration architecture implements automated pipelines that monitor designated directories for new files, validate file structure and content, transform external formats into ADMS representations, load data into operational databases, archive successfully processed files, and alert operators of processing failures. This end-to-end automation minimizes manual intervention while maintaining auditability and recoverability.

## Network Model Import/Export

### CIM XML Format

The Common Information Model (CIM) defines standardized classes and relationships for power system equipment, enabling interoperability between utility applications. CIM XML serializes this model in XML format following industry standards (IEC 61970/61968). ADMS imports CIM XML files containing complete network models or incremental updates from source systems like GIS or network analysis tools.

CIM XML files contain hierarchical equipment definitions with attributes and associations. For example, a transformer is represented with Terminal objects defining connection points, PowerTransformerEnd objects specifying winding parameters, and associations linking the transformer to its containing Substation and connected ConnectivityNode objects. The rich semantic model captures both physical equipment and logical connectivity.

Parsing CIM XML requires handling large file sizes (hundreds of megabytes for utility-scale networks), validating against CIM schema definitions (XSD), resolving cross-references between objects, and mapping CIM classes to ADMS internal database schemas. Incremental processing modes import only changed equipment to reduce processing time for frequent updates.

### MultiSpeak Integration

MultiSpeak is a North American utility integration standard providing XML schemas for common utility data exchanges. MultiSpeak defines message types for network model (CD_Server), SCADA (OA_Server), outage management (OD_Server), and other functional areas. Files conform to MultiSpeak XSD schemas with version-specific message structures.

ADMS processes MultiSpeak files for equipment inventory updates, work order coordination, and outage information exchange. The standard's prescriptive schemas simplify mapping compared to CIM's broader and more complex model. However, MultiSpeak's North American focus and legacy structure make it less suitable for international deployments or modern microservice architectures.

### CSV Custom Formats

For simpler integrations or legacy system compatibility, CSV (Comma-Separated Values) formats provide straightforward data exchange. Custom CSV formats define column layouts, data types, delimiters, and value encodings specific to integration requirements. ADMS implements configurable parsers that adapt to different CSV schemas without code changes.

CSV files work well for tabular data like equipment lists, measurement histories, or customer account information. However, CSV lacks standardization for complex hierarchical relationships, requiring multiple coordinated files or denormalized flat structures. Robust CSV processing handles variations in delimiters (comma, tab, pipe), quoted fields with embedded delimiters, and character encodings (UTF-8, Latin-1).

## File Processing Pipeline

### File Monitoring

File monitoring subsystems detect new files arriving in designated directories using filesystem watchers or periodic directory scans. On Linux, inotify provides efficient event-driven file detection. On Windows, FileSystemWatcher monitors directory changes. For network file shares (SMB, NFS), polling with configurable intervals (1-60 seconds) detects new files.

File naming conventions enable routing to appropriate processors: `NetworkModel_YYYYMMDD_HHMMSS.xml` for CIM imports, `Measurements_*.csv` for historical data. Lock files or atomic rename operations prevent processing incomplete files still being written. Duplicate detection based on filename patterns or content hashes prevents reprocessing identical files.

### Validation

File validation occurs before import processing to detect format errors, missing required fields, invalid data values, and business rule violations. Validation includes:

- **Structural Validation**: XML schema validation (XSD), JSON schema validation, CSV column count and types
- **Referential Integrity**: Foreign key references exist, parent-child relationships are valid
- **Business Rules**: Required attributes are present, enumerated values are valid, numeric ranges are appropriate
- **Consistency Checks**: Equipment connectivity forms valid topology, timestamps are reasonable

Validation failures generate detailed error reports identifying specific problems (line numbers, field names, error descriptions) to facilitate source system corrections. Configurable error thresholds determine whether files with minor issues can proceed with warnings or must be rejected entirely.

### Import Processing

Successful validation triggers import processing that transforms file contents into database operations. Processing strategies include:

- **Full Refresh**: Delete existing data and replace with file contents (appropriate for complete daily network model updates)
- **Incremental Update**: Insert new records, update modified records, delete removed records (requires change detection via timestamps or status flags)
- **Merge**: Combine file data with existing data using conflict resolution rules (last-write-wins, manual review)

Transactional processing ensures atomicity: either all file content imports successfully or the entire import rolls back on error. This prevents partial updates that could corrupt the network model or leave data in inconsistent states. Large files use batched commits (1000-10000 records per transaction) to balance atomicity with memory usage.

Error handling preserves partially valid data when appropriate. If a file contains 10,000 equipment records and 5 have validation errors, the system can import the valid 9,995 while logging failures for correction. Alternatively, strict mode rejects the entire file if any records fail, ensuring complete consistency.

## Scheduling

### Cron Jobs

Scheduled file processing uses cron (Linux) or Task Scheduler (Windows) to trigger imports at defined times. Daily network model imports run during low-activity periods (midnight-4am) to minimize operational impact. Hourly measurement imports process recent historical data for analytics applications.

Cron expressions provide flexible scheduling: `0 2 * * *` runs daily at 2am, `0 */4 * * *` runs every 4 hours, `0 9 * * 1` runs Mondays at 9am. Scheduling considers source system data availability, ADMS maintenance windows, and operational staffing patterns.

### Manual Triggers

Operators can trigger file processing on-demand for ad-hoc data updates or emergency corrections. Web UI provides file upload capability with real-time processing status. REST API endpoints enable programmatic triggering from external systems or scripts. Manual processing respects the same validation and error handling rules as scheduled processing.

Priority queues enable expedited processing of urgent files without waiting for scheduled runs. Emergency network model updates can be processed immediately while routine historical data imports queue for off-peak processing.

## Code Examples

### CIM XML Parser

```java
@Service
public class CIMXMLParser {

    @Autowired
    private NetworkModelService networkModelService;

    public ImportResult importCIMXML(Path xmlFile) throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        factory.setNamespaceAware(true);
        factory.setSchema(loadCIMSchema());

        DocumentBuilder builder = factory.newDocumentBuilder();
        builder.setErrorHandler(new ValidationErrorHandler());

        Document doc = builder.parse(xmlFile.toFile());
        Element root = doc.getDocumentElement();

        ImportResult result = new ImportResult();

        // Parse substations
        NodeList substations = root.getElementsByTagNameNS(CIM_NS, "Substation");
        for (int i = 0; i < substations.getLength(); i++) {
            Element substation = (Element) substations.item(i);
            String id = substation.getAttribute("rdf:ID");
            String name = getChildElementText(substation, "cim:IdentifiedObject.name");

            Substation entity = Substation.builder()
                .id(id)
                .name(name)
                .build();

            networkModelService.saveSubstation(entity);
            result.incrementImported();
        }

        // Parse power transformers
        NodeList transformers = root.getElementsByTagNameNS(CIM_NS, "PowerTransformer");
        for (int i = 0; i < transformers.getLength(); i++) {
            Element transformer = (Element) transformers.item(i);
            PowerTransformer entity = parsePowerTransformer(transformer);
            networkModelService.saveTransformer(entity);
            result.incrementImported();
        }

        return result;
    }

    private String getChildElementText(Element parent, String tagName) {
        NodeList nodes = parent.getElementsByTagName(tagName);
        if (nodes.getLength() > 0) {
            return nodes.item(0).getTextContent();
        }
        return null;
    }
}
```

### CSV File Processor

```python
import csv
import logging
from datetime import datetime
from pathlib import Path

class CSVEquipmentImporter:
    def __init__(self, database):
        self.db = database
        self.logger = logging.getLogger(__name__)

    def process_file(self, csv_path: Path):
        results = {'imported': 0, 'failed': 0, 'errors': []}

        try:
            with open(csv_path, 'r', encoding='utf-8') as file:
                reader = csv.DictReader(file)

                # Validate required columns
                required_cols = ['equipment_id', 'equipment_type', 'name', 'voltage']
                if not all(col in reader.fieldnames for col in required_cols):
                    raise ValueError(f"Missing required columns: {required_cols}")

                batch = []
                for row_num, row in enumerate(reader, start=2):
                    try:
                        equipment = self.validate_and_transform(row, row_num)
                        batch.append(equipment)

                        if len(batch) >= 1000:
                            self.import_batch(batch)
                            results['imported'] += len(batch)
                            batch = []

                    except ValueError as e:
                        results['failed'] += 1
                        results['errors'].append(f"Row {row_num}: {str(e)}")

                # Import remaining records
                if batch:
                    self.import_batch(batch)
                    results['imported'] += len(batch)

        except Exception as e:
            self.logger.error(f"File processing failed: {e}")
            raise

        finally:
            # Archive processed file
            self.archive_file(csv_path, results['failed'] == 0)

        return results

    def validate_and_transform(self, row, row_num):
        equipment_id = row['equipment_id'].strip()
        if not equipment_id:
            raise ValueError("equipment_id is required")

        voltage = float(row['voltage'])
        if voltage <= 0 or voltage > 500:
            raise ValueError(f"Invalid voltage: {voltage}")

        return {
            'id': equipment_id,
            'type': row['equipment_type'],
            'name': row['name'],
            'voltage': voltage,
            'imported_at': datetime.now()
        }

    def import_batch(self, equipment_list):
        self.db.bulk_insert('equipment', equipment_list)

    def archive_file(self, file_path, success):
        archive_dir = file_path.parent / ('archive' if success else 'failed')
        archive_dir.mkdir(exist_ok=True)
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        new_name = f"{file_path.stem}_{timestamp}{file_path.suffix}"
        file_path.rename(archive_dir / new_name)
```

### File Watcher Service

```java
@Service
public class FileWatcherService {

    @Value("${file-import.watch-directory}")
    private String watchDirectory;

    @Autowired
    private FileProcessorFactory processorFactory;

    private WatchService watchService;

    @PostConstruct
    public void initializeWatcher() throws IOException {
        watchService = FileSystems.getDefault().newWatchService();
        Path path = Paths.get(watchDirectory);

        path.register(
            watchService,
            StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_MODIFY
        );

        // Start watcher thread
        Thread watchThread = new Thread(this::watchForFiles);
        watchThread.setDaemon(true);
        watchThread.start();

        logger.info("File watcher started for directory: {}", watchDirectory);
    }

    private void watchForFiles() {
        while (true) {
            try {
                WatchKey key = watchService.take();

                for (WatchEvent<?> event : key.pollEvents()) {
                    if (event.kind() == StandardWatchEventKinds.ENTRY_CREATE) {
                        Path filePath = (Path) event.context();
                        processFile(filePath);
                    }
                }

                key.reset();
            } catch (InterruptedException e) {
                logger.error("File watcher interrupted", e);
                break;
            }
        }
    }

    private void processFile(Path filePath) {
        try {
            // Wait for file write to complete
            Thread.sleep(1000);

            FileProcessor processor = processorFactory.getProcessor(filePath);
            ImportResult result = processor.process(filePath);

            logger.info("File processed: {} - Imported: {}, Failed: {}",
                filePath, result.getImported(), result.getFailed());

        } catch (Exception e) {
            logger.error("Failed to process file: {}", filePath, e);
        }
    }
}
```

## Best Practices

- Implement atomic file operations using temporary files and atomic renames to prevent processing partial writes
- Validate files thoroughly before import to detect and reject malformed data early
- Use transactional processing to ensure atomicity and enable rollback on errors
- Archive successfully processed files with timestamps for audit trails and reprocessing capability
- Generate detailed import reports identifying successes, warnings, and failures with line numbers and field names
- Implement retry logic with exponential backoff for transient failures (network issues, database locks)
- Monitor file processing latency and alert when files aren't processed within expected timeframes
- Configure appropriate file retention policies balancing storage costs with audit requirements
- Document file format specifications including schemas, example files, and data dictionaries
- Test file processing with edge cases: empty files, maximum sizes, special characters, and malformed data
