# Ansible Bird Role


## Description

Ansible role which installs and configures [Bird](https://bird.network.cz/).

## Role Variables

Default Bird configuration
```yml
bird_config_bgp_asn_local: 64512

bird_config_default:
  protocols:
    direct: |
      interface "eth1" "eth2*";
    kernel: |
      persist;
      scan time 20;
      import all;
      export all;
    device: |
      scan time 5;
```

To configure all peers you may want to dynamically build a dict...
```yml
- name: ansible | render bird config for VPN
  vars:
    bird_config_peer_bgp_name: "{{ vpn_connection_config.0.vpn_connection['@id'] | replace('-', '') | upper }}_{{ vpn_connection_index }}"
    bird_config_peer_bgp_asn_remote: "{{ vpn_connection_config.1.vpn_gateway.bgp.asn }}"
    bird_config_peer_bgp_ip_remote: "{{ vpn_connection_config.1.vpn_gateway.tunnel_inside_address.ip_address }}"
    bird_config_peer_bgp_ip_local: "{{ vpn_connection_config.1.customer_gateway.tunnel_inside_address.ip_address }}"
  set_fact:
    bird_config_peers: "{{ bird_config_peers | combine(lookup('template', 'templates/bird_config_peer.j2'))  }}"
  loop: "{{ query('subelements', vpn_connection_configs, 'vpn_connection.ipsec_tunnel', {'skip_missing': true}) }}"
  loop_control:
    loop_var: vpn_connection_config
    index_var: vpn_connection_index

- name: ansible | render bird config for DX
  vars:
    interface_name: >-
      {%- if dx_connection_interface is defined -%}
      {{ dx_connection_interface }}.{{ dx_connection_config.0.vlan }}
      {%- else -%}
      {{ bird_interface_vlan_dict[dx_connection_config.0.vlan] }}
      {%- endif -%}
    bird_config_peer_bgp_name: "VLAN{{ dx_connection_config.0.vlan }}"
    bird_config_peer_bgp_asn_remote: "{{ dx_connection_config.0.amazonSideAsn }}"
    bird_config_peer_bgp_ip_remote: "{{ dx_connection_config.0.amazonAddress | ipaddr('address') }}"
    bird_config_peer_bgp_ip_local: "{{ dx_connection_config.0.customerAddress }}"
    bird_config_peer_bgp_authkey: "{{ dx_connection_config.1.authKey }}"
    bird_config_peer_bfd_interface: "{{ interface_name }}"
  set_fact:
    bird_config_peers: "{{ bird_config_peers | combine(lookup('template', 'templates/bird_config_peer.j2'))  }}"
  loop: "{{ query('subelements', dx_connection_configs, 'bgpPeers', {'skip_missing': true}) }}"
  loop_control:
    loop_var: dx_connection_config
```

... with at least this Jinja template
```jinja
{
  "{{ bird_config_peer_bgp_name }}": {
    "proto": "bgp",
    "bgp_asn_remote": "{{ bird_config_peer_bgp_asn_remote }}",
    "bgp_ip_remote": "{{ bird_config_peer_bgp_ip_remote }}",
    "bgp_ip_local": "{{ bird_config_peer_bgp_ip_local }}",
    "bgp_authkey": "{{ bird_config_peer_bgp_authkey | default() }}",
    "bfd_interface": "{{ bird_config_peer_bfd_interface | default() }}"
  }
}
```
