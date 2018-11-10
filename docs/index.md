# XCRI Documentation

Welcome to the main documentation site for the XSEDE
Cyberinfrastructure Resource Integration team. Here, you'll
find descriptions and documentation for the various toolkits 
we provide to computational resource providers and users. 

## Toolkits
 * The XSEDE-Compatible Basic Cluster (XCBC) software toolkit 
enables campus CI resource administrators to build a local 
cluster from scratch, which is then easily interoperable with 
XSEDE-supported CI resources. XCBC is very simple in concept: 
pull the lever, have a cluster built for you complete with an 
open source resource manager / scheduler and all of the essential 
tools needed to run a cluster, and have those tools set in place 
in ways that mimic the basic setup on an XSEDE-supported cluster
. The XCBC is based on the OpenHPC project, and consists of XSEDE
-developed Ansible playbooks and templates designed to ease the 
work required to build a cluster. Consult the XSEDE Knowledge Base 
for complete information about how to use XCBC to set up a cluster.

 * The XSEDE National Integration Toolkit (XNIT). Suppose you 
already have a cluster that you are happy with and you want to 
add too it software tools that will allow users to use open sources 
software like that on XSEDE, or other particular pieces of software 
that you think are important, but you don't want to blow up your cluster 
to add that capability? XNIT is for you. You can add all of the basic software 
that is in SCBC, as relocatable RPMs (Resource Package Manger), via a YUM repo
. (YUM Stands for Yellowdog Updater, Modified). The RPMs in XNIT allow you to 
expand the functionality of your cluster, in ways that mimic the setup on an 
XSEDE cluster. XNIT packages include specific scientific, mathematical, and 
visualization applications that have been useful on XSEDE systems. Systems 
administrators may pick and choose what they want to add to their local cluster
; updates may be configured to run automatically or manually. Currently the XNIT 
repository is available for x86_64 systems running CentOS 6 or 7. Consult the XSEDE 
Knowledge Base for more information.

 * XCRI offers a Cluster Monitoring toolkit - compatible with the 
XCBC or any OpenHPC cluster running Warewulf, that allows sites to 
install tools that monitor both cluster health and job statistics. 
The toolkit allows administrators to generate fine-grained reports 
of usage based on users, projects, and job types, which can be a great 
aid in keeping track of ROI or justifying future funding.

## Site Visits
XCRI staff will travel in person to your campus to help 
implement XNIT, XCBC, or any other XCRI tools on your campus
. After an initial phone consultation, we can assist onsite 
with configuration and ensure that you have the knowledge that 
you need to maintain your system. That's right…. XSEDE will pay 
to fly XSEDE staff to your campus and help you with your campus 
cluster, even if you have no particular relationship to XSEDE. 
You can see the list of places we have gone to give talks or help 
people set up clusters. You can read detailed descriptions of past 
visits to campuses to help with local clusters in the XSEDE 2016 paper 
Implementation of Simple XSEDE-Like Clusters: Science Enabled and Lessons 
Learned. These site visits are funded by XSEDE, including staff travel and 
lodgings.

## Presentations
Slides from our SC18 booth presentation [are available here](https://github.com/XSEDE/XCRI-Docs/raw/master/files/SC18-XCRI-Booth-Talk.pdf).
