---
title: iOS中将数字用符合隔开(比如逗号) # 这是标题
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
这次项目中遇到一个需求，将数字每隔3个用逗号隔开。因为如果数字太大确实看着有点眼花。
　　所以就想着写了一个NSString的category，用起来效果还不错，代码如下：
### .h的声明
    #import <Foundation/Foundation.h>

    @interface NSString (NumberSplitWithComma)
    /**
     *数字以逗号隔开 例：123，321.11
     */
    - (NSString *)ld_numberSplitWithComma;
    /**
     *@param puctutation 符合
     *数字以某个符合隔开 例：123，321.11
     */
    - (NSString *)ld_numberSplitWithPunctutaion:(NSString *)puctutation;
    @end
### .m文件的实现
    #import "NSString+NumberSplitWithComma.h"

    @implementation NSString (NumberSplitWithComma)

    - (NSString *)ld_numberSplitWithComma{
        if ([self private_isNumber]) {
            NSMutableString *muString = [NSMutableString tringWithString:self];
            return [self private_insert:muString withPunctuation:@","];
        }else{
            return self;
        }
    }
    - (NSString *)ld_numberSplitWithPunctutaion:(NSString *)puctutation{
        if ([self private_isNumber]) {
             NSMutableString *muString = [NSMutableString stringWithString:self];
             return [self private_insert:muString withPunctuation:puctutation];
         }else{
             return self;
         }
    }
    - (NSMutableString *)private_insert:(NSMutableString *)string withPunctuation:(NSString *)punctuation{
        NSUInteger maxLength = string.length;
        if ([string containsString:punctuation]) {
            maxLength = [string rangeOfString:punctuation].location;
        }else if ([string containsString:@"."]){
            maxLength = [string rangeOfString:@"."].location;
        }
        if (maxLength-([string containsString:@"-"]?1:0)>3) {
            [string insertString:punctuation atIndex:(maxLength-3)];
            [self private_insert:string withPunctuation:punctuation];
        }else{
            return string;
        }
        return string;
    }
     - (Boolean)private_isNumber{
        NSPredicate *predicate = [NSPredicate predicateWithFormat:@"SELF MATCHES %@",@"^[-0-9.]*$"];
        if ([predicate evaluateWithObject:self]) {
            return YES;
         }
        return NO;
     }

# 补充
　　有简友反应可以使用NSNumberFormatter，然后研究了一番，确实能够很方便的达到效果。
1、通过使用formatter style，苹果帮我们定义好的格式。

    NSNumberFormatter *numberFormatter =   [[NSNumberFormatter alloc] init];
    [numberFormatter setNumberStyle:NSNumberFormatterDecimalStyle];
    NSString *formattedNumberString = [numberFormatter stringFromNumber:@122344.4563];
    NSLog(@"formattedNumberString: %@", formattedNumberString);
    // Output for locale en_US: "formattedNumberString: formattedNumberString: 122,344.453"
NSNumberFormatter 的numberStyle是一个枚举

         typedef NS_ENUM(NSUInteger, NSNumberFormatterStyle) {
     NSNumberFormatterNoStyle = kCFNumberFormatterNoStyle,四舍五入
     NSNumberFormatterDecimalStyle = kCFNumberFormatterDecimalStyle,金额 100,200,300.123
     NSNumberFormatterCurrencyStyle = kCFNumberFormatterCurrencyStyle,货币 $100,200,300.12
     NSNumberFormatterPercentStyle = kCFNumberFormatterPercentStyle,百分比 12%
     NSNumberFormatterScientificStyle = kCFNumberFormatterScientificStyle,科学计数法 1.234E8
     NSNumberFormatterSpellOutStyle = kCFNumberFormatterSpellOutStyle,口语 One...
     NSNumberFormatterOrdinalStyle NS_ENUM_AVAILABLE(10_11, 9_0) = kCFNumberFormatterOrdinalStyle,
     NSNumberFormatterCurrencyISOCodeStyle NS_ENUM_AVAILABLE(10_11, 9_0) = kCFNumberFormatterCurrencyISOCodeStyle,
     NSNumberFormatterCurrencyPluralStyle NS_ENUM_AVAILABLE(10_11, 9_0) = kCFNumberFormatterCurrencyPluralStyle,
     NSNumberFormatterCurrencyAccountingStyle NS_ENUM_AVAILABLE(10_11, 9_0) = kCFNumberFormatterCurrencyAccountingStyle,
     };
２、通过一个格式字符串自定义格式

    NSNumberFormatter *numberFormatter = [[NSNumberFormatter alloc] init];
    [numberFormatter setPositiveFormat:@"###0.##"];
    NSString *formattedNumberString = [numberFormatter stringFromNumber:@122344.4563];
    NSLog(@"formattedNumberString: %@", formattedNumberString);
    // Output for locale en_US: "formattedNumberString: formattedNumberString: 122,344.45"
