# VoiceAttack phrases — VAICOM digit-run fix

Paste each into the command's "When I say" field, replacing what is there.
Both require the patched VAICOMPRO.dll.

## AIRIO - Set Laser Code

The spaced form keeps paced delivery working. The 192 literals are the exact
valid codes — unlike [511..788], which also generates 590, 699, 780 and other
combinations the thumbwheels cannot produce.

Laser Code [5..7] [1..8] [1..8];Laser Code 511;Laser Code 512;Laser Code 513;Laser Code 514;Laser Code 515;Laser Code 516;Laser Code 517;Laser Code 518;Laser Code 521;Laser Code 522;Laser Code 523;Laser Code 524;Laser Code 525;Laser Code 526;Laser Code 527;Laser Code 528;Laser Code 531;Laser Code 532;Laser Code 533;Laser Code 534;Laser Code 535;Laser Code 536;Laser Code 537;Laser Code 538;Laser Code 541;Laser Code 542;Laser Code 543;Laser Code 544;Laser Code 545;Laser Code 546;Laser Code 547;Laser Code 548;Laser Code 551;Laser Code 552;Laser Code 553;Laser Code 554;Laser Code 555;Laser Code 556;Laser Code 557;Laser Code 558;Laser Code 561;Laser Code 562;Laser Code 563;Laser Code 564;Laser Code 565;Laser Code 566;Laser Code 567;Laser Code 568;Laser Code 571;Laser Code 572;Laser Code 573;Laser Code 574;Laser Code 575;Laser Code 576;Laser Code 577;Laser Code 578;Laser Code 581;Laser Code 582;Laser Code 583;Laser Code 584;Laser Code 585;Laser Code 586;Laser Code 587;Laser Code 588;Laser Code 611;Laser Code 612;Laser Code 613;Laser Code 614;Laser Code 615;Laser Code 616;Laser Code 617;Laser Code 618;Laser Code 621;Laser Code 622;Laser Code 623;Laser Code 624;Laser Code 625;Laser Code 626;Laser Code 627;Laser Code 628;Laser Code 631;Laser Code 632;Laser Code 633;Laser Code 634;Laser Code 635;Laser Code 636;Laser Code 637;Laser Code 638;Laser Code 641;Laser Code 642;Laser Code 643;Laser Code 644;Laser Code 645;Laser Code 646;Laser Code 647;Laser Code 648;Laser Code 651;Laser Code 652;Laser Code 653;Laser Code 654;Laser Code 655;Laser Code 656;Laser Code 657;Laser Code 658;Laser Code 661;Laser Code 662;Laser Code 663;Laser Code 664;Laser Code 665;Laser Code 666;Laser Code 667;Laser Code 668;Laser Code 671;Laser Code 672;Laser Code 673;Laser Code 674;Laser Code 675;Laser Code 676;Laser Code 677;Laser Code 678;Laser Code 681;Laser Code 682;Laser Code 683;Laser Code 684;Laser Code 685;Laser Code 686;Laser Code 687;Laser Code 688;Laser Code 711;Laser Code 712;Laser Code 713;Laser Code 714;Laser Code 715;Laser Code 716;Laser Code 717;Laser Code 718;Laser Code 721;Laser Code 722;Laser Code 723;Laser Code 724;Laser Code 725;Laser Code 726;Laser Code 727;Laser Code 728;Laser Code 731;Laser Code 732;Laser Code 733;Laser Code 734;Laser Code 735;Laser Code 736;Laser Code 737;Laser Code 738;Laser Code 741;Laser Code 742;Laser Code 743;Laser Code 744;Laser Code 745;Laser Code 746;Laser Code 747;Laser Code 748;Laser Code 751;Laser Code 752;Laser Code 753;Laser Code 754;Laser Code 755;Laser Code 756;Laser Code 757;Laser Code 758;Laser Code 761;Laser Code 762;Laser Code 763;Laser Code 764;Laser Code 765;Laser Code 766;Laser Code 767;Laser Code 768;Laser Code 771;Laser Code 772;Laser Code 773;Laser Code 774;Laser Code 775;Laser Code 776;Laser Code 777;Laser Code 778;Laser Code 781;Laser Code 782;Laser Code 783;Laser Code 784;Laser Code 785;Laser Code 786;Laser Code 787;Laser Code 788

## AIRIO - Datalink manual tuning

First form is the existing phrase, kept so paced delivery still matches.
Second lets you speak the frequency with its fixed leading 3:
"link tune three zero five five" sets wheels 0, 5, 5 (305.50).

Link Tune [0..9] [0..9] decimal [0..9];Link Tune [3000..3999]

## Notes

- Say "laser code five seven seven" naturally; the engine collapses it to 577
  and the literal matches.
- The four-digit form "laser code one five seven seven" also works — the
  handler strips the implicit leading 1.
- A collapsed "99.9" still cannot be matched by any range-based phrase, because
  VoiceAttack cannot generate a string containing a period. Say the frequency
  instead.
