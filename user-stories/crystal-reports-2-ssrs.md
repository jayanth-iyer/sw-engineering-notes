# Spec-to-RDL Generator (Legacy Migration Agent)
**BDD Acceptance Tests & User Stories**

---

## Overview
This specification defines the behavior for the **Spec-to-RDL Generator**, an AI-assisted migration agent that converts legacy Crystal Reports metadata into valid SSRS `.rdl` XML definitions with automated schema validation and self-correction.

---

## Feature Specification

```gherkin
Feature: Spec-to-RDL Report Migration Engine
  As a Migration Engineer
  I want an automated pipeline to parse Crystal Reports metadata, translate formulas, and synthesize valid SSRS RDL files
  So that legacy report migrations are automated with zero manual XML syntax errors.

  Background:
    Given the migration workspace is initialized on a local development machine
    And Python 3.10+ with "xml.etree.ElementTree" is available for local validation

  # ---------------------------------------------------------------------------
  # User Story 1: Metadata Extraction
  # ---------------------------------------------------------------------------
  @extraction @csharp
  Scenario: Successfully extract metadata from a compiled Crystal Report (.rpt) file
    Given I have a valid Crystal Report binary file "SalesSummary.rpt"
    When I execute the C# metadata extraction utility on "SalesSummary.rpt"
    Then the system should produce a normalized JSON file "SalesSummary_metadata.json"
    And "SalesSummary_metadata.json" must contain the following keys:
      | Key               | Expected Content                              |
      | data_source       | SQL query string or stored procedure reference |
      | parameters        | Parameter names, data types, and default values|
      | formulas          | Array of raw Crystal/Basic formula expressions|
      | layout            | Grouping hierarchy and band positioning data  |

  # ---------------------------------------------------------------------------
  # User Story 2: Formula Translation
  # ---------------------------------------------------------------------------
  @translation @ollama @qwen
  Scenario Outline: Translate Crystal formula expressions into SSRS Visual Basic syntax
    Given I have a raw Crystal formula "<CrystalSyntax>"
    When I send the formula to the local translation engine powered by "qwen2.5-coder:3b"
    Then the returned response should be an SSRS-compliant VB expression matching "<VBSyntax>"

    Examples:
      | CrystalSyntax                                    | VBSyntax                                        |
      | If {Orders.Amount} > 1000 Then "High" Else "Low" | =IIf(Fields!Amount.Value > 1000, "High", "Low") |
      | IsNull({Orders.ShipDate})                        | =IsNothing(Fields!ShipDate.Value)               |
      | ToText({Orders.ID})                              | =CStr(Fields!ID.Value)                          |
      | CurrentDate                                      | =Today()                                        |

  # ---------------------------------------------------------------------------
  # User Story 3: RDL XML Synthesis
  # ---------------------------------------------------------------------------
  @synthesis @cursor-agent
  Scenario: Synthesize a full SSRS RDL XML file from translated metadata
    Given I have a valid "translated_metadata.json" file
    When I trigger the Cursor Agent with the prompt "Generate output.rdl based on translated_metadata.json"
    Then a file named "output.rdl" should be created in the current directory
    And "output.rdl" must contain the following mandatory RDL XML nodes:
      | Required Node       |
      | <Report>            |
      | <DataSources>       |
      | <DataSets>          |
      | <ReportParameters>  |
      | <Body>              |
      | <Tablix>            |

  # ---------------------------------------------------------------------------
  # User Story 4: Automated Self-Correction Loop
  # ---------------------------------------------------------------------------
  @validation @reflection-loop
  Scenario: Automatically detect invalid XML syntax and auto-repair via terminal feedback
    Given the Cursor Agent generates an "output.rdl" file containing an unclosed XML tag at line 35
    When the Cursor Agent executes "python validate.py output.rdl" in the integrated terminal
    Then the command should fail with exit status 1
    And the stdout should print an "XML ParseError" message
    When the Cursor Agent reads the error output from the terminal execution
    Then the Agent should rewrite "output.rdl" to fix the structural syntax error
    And re-running "python validate.py output.rdl" should exit with status 0
    And stdout should display "SUCCESS: Valid XML Schema!"