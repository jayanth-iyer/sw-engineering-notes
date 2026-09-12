# MCP Gateway (Enterprise Governance & Context Proxy)
**Complete Technical Architecture, BDD Specifications & Local Testing Guide**

---

## 1. Executive Summary

The **MCP Gateway** acts as a secure, centralized proxy between client-side developer tools (such as Cursor or IDE agents) and enterprise backend infrastructure (internal databases, APIs, security scanners, and vector search).

By governing tool calls in flight, the Gateway enforces identity-based access control, intercepts and redacts sensitive data (PII/MNPI), and maintains WORM-compliant audit logs before context ever leaves the secure perimeter.

---

## 2. Feature Specification (BDD User Stories)

```gherkin
Feature: MCP Gateway Security & Governance Proxy
  As an Enterprise Security Engineer
  I want an MCP Gateway sitting between local developer tools and downstream tools
  So that PII/MNPI is masked, administrative tool calls are audited, and unauthorized actions are blocked.

  Background:
    Given the MCP Gateway service is running locally on "http://localhost:8000"
    And the Gateway is connected to an upstream SQLite audit log database

  # ---------------------------------------------------------------------------
  # User Story 1: Dynamic Tool Discovery Based on Auth Context
  # ---------------------------------------------------------------------------
  @discovery @mcp-protocol
  Scenario: Authenticated developer discovers allowed enterprise tools
    Given a client sends a JSON-RPC "tools/list" request with a valid API token for user "dev_user"
    When the Gateway processes the discovery request
    Then the Gateway should return a list of tools including "query_database" and "run_security_scan"
    And sensitive administrative tools like "deploy_to_prod" should be excluded from the list

  # ---------------------------------------------------------------------------
  # User Story 2: In-Flight PII and Sensitive Data Redaction
  # ---------------------------------------------------------------------------
  @governance @redaction
  Scenario Outline: Redact sensitive information from incoming prompts before forwarding
    Given a client invokes tool "query_database" with prompt payload "<RawPayload>"
    When the Gateway governance middleware evaluates the request payload
    Then the payload passed to the downstream tool must match "<SanitizedPayload>"

    Examples:
      | RawPayload                                                  | SanitizedPayload                                                 |
      | Query user where ssn = '999-00-1234'                        | Query user where ssn = '[REDACTED_PII]'                          |
      | SELECT * FROM accounts WHERE api_key = 'sk_live_abc123'     | SELECT * FROM accounts WHERE api_key = '[REDACTED_CREDENTIAL]'   |

  # ---------------------------------------------------------------------------
  # User Story 3: Access Control & Authorization Enforcement (AXA)
  # ---------------------------------------------------------------------------
  @authorization @security
  Scenario: Block unauthorized or high-risk tool execution
    Given a client issues an MCP tool call "execute_db_migration"
    And user "dev_user" does not hold the "Admin" role
    When the Gateway interceptor checks permission policies
    Then the request should be rejected with JSON-RPC error code -32001
    And the response message should state "Access Denied: High-risk action requires human approval."

  # ---------------------------------------------------------------------------
  # User Story 4: WORM-Compliant Audit Logging (AXT)
  # ---------------------------------------------------------------------------
  @audit @compliance
  Scenario: Record tamper-proof audit trail for invoked tools
    Given user "dev_user" successfully invokes tool "run_security_scan"
    When the tool execution completes with status "200 OK"
    Then an audit record should be written to the local SQLite audit table
    And the record must contain "timestamp", "user_id", "tool_name", "prompt_hash", and "execution_latency_ms"