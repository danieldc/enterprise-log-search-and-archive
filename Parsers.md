# Introduction #

These rules are contributed by the ELSA community.

# Cisco NAT #
| Author | C Mallow |
|:-------|:---------|
| New classes | FIREWALL\_STATIC\_NAT, FIREWALL\_DYNAMIC\_NAT |
| New fields | natip, natport |

## Ruleset ##
```
<ruleset name="Cisco NAT">
  <pattern>%</pattern>
<rule provider="ELSA" class='MUST_MATCH_DB' id='MUST_MATCH_DB'>
                <patterns>
                    <pattern>Built static translation from @ESTRING:s0:::@@IPv4:i0:@ to @ESTRING:s1:::@@IPv4:i1:@</pattern>
                    <pattern>Teardown static translation from @ESTRING:s0:::@@IPv4:i0:@ to @ESTRING:s1:::@@IPv4:i1:@ duration @ANYSTRING::@</pattern>
                 </patterns>
                <examples>
                    <example>
                        <test_message program="%FWSM-6-305009">Built static translation from INSIDE:10.1.1.1 to OUTSIDE:1.2.3.4</test_message>
                        <test_values>
                            <test_value name="i0">10.1.1.1</test_value>
                            <test_value name="s0">INSIDE</test_value>
                            <test_value name="i1">1.2.3.4</test_value>
                            <test_value name="s1">OUTSIDE</test_value>
                        </test_values>
                    </example>
                </examples>
            </rule>
            <rule provider="ELSA" class='MUST_MATCH_DB' id='MUST_MATCH_DB'>
                <patterns>
                    <pattern>Built dynamic@QSTRING:i4: @translation from @ESTRING:s1:::@@IPv4:i0:@/@NUMBER:i1:@ to @ESTRING:s2:::@@IPv4:i2:@/@NUMBER:i3:@</pattern>
                    <pattern>Teardown dynamic@QSTRING:i4: @translation from @ESTRING:s1:::@@IPv4:i0:@/@NUMBER:i1:@ to @ESTRING:s2:::@@IPv4:i2:@/@NUMBER:i3:@ duration @ANYSTRING::@</pattern>
                 </patterns>
                <examples>
                    <example>
                        <test_message program="%FWSM-6-305012">Teardown dynamic tcp translation from INSIDE:10.1.1.1/3456 to OUTSIDE:1.2.3.4/4567 duration 0:00:30</test_message>
                        <test_values>
                            <test_value name="i4">tcp</test_value>
                            <test_value name="s1">INSIDE</test_value>
                            <test_value name="i0">10.1.1.1</test_value>
                            <test_value name="i1">3456</test_value>
                            <test_value name="s2">OUTSIDE</test_value>
                            <test_value name="i2">1.2.3.4</test_value>
                            <test_value name="i3">4567</test_value>                            
                        </test_values>
                    </example>
                </examples>
            </rule>
</ruleset>
```

## Schema ##

```
INSERT INTO fields (field, field_type, pattern_type) VALUES ("natip", "int", "IPv4");
INSERT INTO fields (field, field_type, pattern_type) VALUES ("natport", "int", "NUMBER");

SELECT MAX(id) INTO @max_id FROM classes;
INSERT INTO classes (id, class) VALUES ((SELECT @max_id+1 FROM classes), "FIREWALL_STATIC_NAT");

INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_STATIC_NAT"), (SELECT id FROM fields WHERE field="srcip", 5);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_STATIC_NAT"), (SELECT id FROM fields WHERE field="natip", 6);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_STATIC_NAT"), (SELECT id FROM fields WHERE field="i_int", 11);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_STATIC_NAT"), (SELECT id FROM fields WHERE field="o_int", 12);

SELECT MAX(id) INTO @max_id FROM classes;
INSERT INTO classes (id, class) VALUES ((SELECT @max_id+1 FROM classes), "FIREWALL_DYNAMIC_NAT");

INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_DYNAMIC_NAT"), (SELECT id FROM fields WHERE field="srcip", 5);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_DYNAMIC_NAT"), (SELECT id FROM fields WHERE field="srcport", 6);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_DYNAMIC_NAT"), (SELECT id FROM fields WHERE field="natip", 7);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_DYNAMIC_NAT"), (SELECT id FROM fields WHERE field="natport", 8);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_DYNAMIC_NAT"), (SELECT id FROM fields WHERE field="proto", 9);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_DYNAMIC_NAT"), (SELECT id FROM fields WHERE field="i_int", 11);
INSERT INTO fields_classes_map (class_id, field_id, field_order) VALUES ((SELECT id FROM classes WHERE class="FIREWALL_DYNAMIC_NAT"), (SELECT id FROM fields WHERE field="o_int", 12);
```