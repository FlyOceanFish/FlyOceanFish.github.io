---
title: iOS之VVeboTableView性能探究之路 # 这是标题
date: 2018-9-04 14:30:00
categories:  # 这里写的分类会自动汇集到 categories 页面上，分类可以多级
- iOS # 一级分类
tags:   # 这里写的标签会自动汇集到 tags 页面上
- iOS
---
# 背景
话说如果项目越来越复杂，页面随之也会变得复杂，最终导致的就是性能优化问题。性能优化方面也是比较多的，我们通过研究`VVeboTableView`这个项目的优化点进而学习优化思路。

其实项目优化最终仅仅就是一个目的，页面流畅度达到60FPS,这个项目通过优化甚至在低端机都表现良好，接下来就让我们深入探究一下它是如何做到的。

不过关于优化一般都是不要过早的大方面优化，最好是项目稳定之后，具体针对某一具体的复杂没有达到60FPS的页面进行优化。
# 探究之路
## 滑动过程中不加载视图
首先是`VVeboTableView`实现了`UIScrollViewDelegate的几个方法`,这些方法中最关键的就是`- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset`这个方法，在我们抬手的那一刻，通过此时手指的位置计算出当前页面需要加载的所有`UITableViewCell`。屏幕外上下各加3个。

```Objective-C
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView{
    [needLoadArr removeAllObjects];
}

//按需加载 - 如果目标行与当前行相差超过指定行数，只在目标滚动范围的前后指定3行加载。
- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset{

    NSIndexPath *ip = [self indexPathForRowAtPoint:CGPointMake(0, targetContentOffset->y)];
    NSIndexPath *cip = [[self indexPathsForVisibleRows] firstObject];
    NSInteger skipCount = 8;
    if (labs(cip.row-ip.row)>skipCount) {
        NSArray *temp = [self indexPathsForRowsInRect:CGRectMake(0, targetContentOffset->y, self.width, self.height)];
        NSMutableArray *arr = [NSMutableArray arrayWithArray:temp];
        if (velocity.y<0) {
            NSIndexPath *indexPath = [temp lastObject];
            if (indexPath.row+3<datas.count) {
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+1 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row+3 inSection:0]];
            }
        } else {
            NSIndexPath *indexPath = [temp firstObject];
            if (indexPath.row>3) {
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-3 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-2 inSection:0]];
                [arr addObject:[NSIndexPath indexPathForRow:indexPath.row-1 inSection:0]];
            }
        }
        [needLoadArr addObjectsFromArray:arr];
    }
}

- (BOOL)scrollViewShouldScrollToTop:(UIScrollView *)scrollView{
    scrollToToping = YES;
    return YES;
}

- (void)scrollViewDidEndScrollingAnimation:(UIScrollView *)scrollView{
    NSLog(@"scrollViewDidEndScrollingAnimation\n");
    scrollToToping = NO;
    [self loadContent];
}

- (void)scrollViewDidScrollToTop:(UIScrollView *)scrollView{
    scrollToToping = NO;
    [self loadContent];
}
```

紧接着我们来看UITableView加载cell的方法，在此方法中是调用了
`[self drawCell:cell withIndexPath:indexPath]`此方法，此方法实现如下：
```Objective-C
- (void)drawCell:(VVeboTableViewCell *)cell withIndexPath:(NSIndexPath *)indexPath{
    NSDictionary *data = [datas objectAtIndex:indexPath.row];
    cell.selectionStyle = UITableViewCellSelectionStyleNone;
    [cell clear];
    cell.data = data;
    if (needLoadArr.count>0&&[needLoadArr indexOfObject:indexPath]==NSNotFound) {
        [cell clear];
        return;
    }
    if (scrollToToping) {
        return;
    }
    [cell draw];
}
```
此方法中通过判断是否需要绘制cell，如果不需要会返回空白。所以在滑动的过程中我们会看到一些空白页面

## 减少视图的层级和数量
```Objective-C
//将主要内容绘制到图片上
- (void)draw{
    if (drawed) {
        return;
    }
    NSInteger flag = drawColorFlag;
    drawed = YES;
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        CGRect rect = [_data[@"frame"] CGRectValue];
        UIGraphicsBeginImageContextWithOptions(rect.size, YES, 0);
        CGContextRef context = UIGraphicsGetCurrentContext();
        [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];
        CGContextFillRect(context, rect);
        if ([_data valueForKey:@"subData"]) {
            [[UIColor colorWithRed:243/255.0 green:243/255.0 blue:243/255.0 alpha:1] set];
            CGRect subFrame = [_data[@"subData"][@"frame"] CGRectValue];
            CGContextFillRect(context, subFrame);
            [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set];
            CGContextFillRect(context, CGRectMake(0, subFrame.origin.y, rect.size.width, .5));
        }

        {
            float leftX = SIZE_GAP_LEFT+SIZE_AVATAR+SIZE_GAP_BIG;
            float x = leftX;
            float y = (SIZE_AVATAR-(SIZE_FONT_NAME+SIZE_FONT_SUBTITLE+6))/2-2+SIZE_GAP_TOP+SIZE_GAP_SMALL-5;
            [_data[@"name"] drawInContext:context withPosition:CGPointMake(x, y) andFont:FontWithSize(SIZE_FONT_NAME)
                             andTextColor:[UIColor colorWithRed:106/255.0 green:140/255.0 blue:181/255.0 alpha:1]
                                andHeight:rect.size.height];
            y += SIZE_FONT_NAME+5;
            float fromX = leftX;
            float size = [UIScreen screenWidth]-leftX;
            NSString *from = [NSString stringWithFormat:@"%@  %@", _data[@"time"], _data[@"from"]];
            [from drawInContext:context withPosition:CGPointMake(fromX, y) andFont:FontWithSize(SIZE_FONT_SUBTITLE)
                   andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1]
                      andHeight:rect.size.height andWidth:size];
        }

        {
            CGRect countRect = CGRectMake(0, rect.size.height-30, [UIScreen screenWidth], 30);
            [[UIColor colorWithRed:250/255.0 green:250/255.0 blue:250/255.0 alpha:1] set];
            CGContextFillRect(context, countRect);
            float alpha = 1;

            float x = [UIScreen screenWidth]-SIZE_GAP_LEFT-10;
            NSString *comments = _data[@"comments"];
            if (comments) {
                CGSize size = [comments sizeWithConstrainedToSize:CGSizeMake(CGFLOAT_MAX, CGFLOAT_MAX) fromFont:FontWithSize(SIZE_FONT_SUBTITLE) lineSpace:5];

                x -= size.width;
                [comments drawInContext:context withPosition:CGPointMake(x, 8+countRect.origin.y)
                                andFont:FontWithSize(12)
                           andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1]
                              andHeight:rect.size.height];
                [[UIImage imageNamed:@"t_comments.png"] drawInRect:CGRectMake(x-5, 10.5+countRect.origin.y, 10, 9) blendMode:kCGBlendModeNormal alpha:alpha];
                commentsRect = CGRectMake(x-5, self.height-50, [UIScreen screenWidth]-x+5, 50);
                x -= 20;
            }

            NSString *reposts = _data[@"reposts"];
            if (reposts) {
                CGSize size = [reposts sizeWithConstrainedToSize:CGSizeMake(CGFLOAT_MAX, CGFLOAT_MAX) fromFont:FontWithSize(SIZE_FONT_SUBTITLE) lineSpace:5];

                x -= MAX(size.width, 5)+SIZE_GAP_BIG;
                [reposts drawInContext:context withPosition:CGPointMake(x, 8+countRect.origin.y)
                                andFont:FontWithSize(12)
                           andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:1]
                              andHeight:rect.size.height];

                [[UIImage imageNamed:@"t_repost.png"] drawInRect:CGRectMake(x-5, 11+countRect.origin.y, 10, 9) blendMode:kCGBlendModeNormal alpha:alpha];
                repostsRect = CGRectMake(x-5, self.height-50, commentsRect.origin.x-x, 50);
                x -= 20;
            }

            [@"•••" drawInContext:context
                     withPosition:CGPointMake(SIZE_GAP_LEFT, 8+countRect.origin.y)
                          andFont:FontWithSize(11)
                     andTextColor:[UIColor colorWithRed:178/255.0 green:178/255.0 blue:178/255.0 alpha:.5]
                        andHeight:rect.size.height];

            if ([_data valueForKey:@"subData"]) {
                [[UIColor colorWithRed:200/255.0 green:200/255.0 blue:200/255.0 alpha:1] set];
                CGContextFillRect(context, CGRectMake(0, rect.size.height-30.5, rect.size.width, .5));
            }
        }

        UIImage *temp = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        dispatch_async(dispatch_get_main_queue(), ^{
            if (flag==drawColorFlag) {
                postBGView.frame = rect;
                postBGView.image = nil;
                postBGView.image = temp;
            }
        });
    });
    [self drawText];
    [self loadThumb];
}
```
通过以上代码我们可以看出，将主要的内容比如名字、评论图标和数量、转发图标和数量等一些内容全部绘制到了一张图片上，然后设置到了`UIImageView`上
## 通过异步绘制视图，主线程进行设置
上边的代码我们看到在绘制的过程中是在全局并发队列中进行的，并且最终回到了主线程进行设置，而且这也是必须要这样做的
```Objective-C
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
  ...
  dispatch_async(dispatch_get_main_queue(), ^{
    if (flag==drawColorFlag) {
        postBGView.frame = rect;
        postBGView.image = nil;
        postBGView.image = temp;
    }
});
}
[self drawText];
[self loadThumb];
```
接下来让我看一下`[self drawText]`方法的实现。
```Objective-C
- (void)drawText{
    if (label==nil||detailLabel==nil) {
        [self addLabel];
    }
    label.frame = [_data[@"textRect"] CGRectValue];
    [label setText:_data[@"text"]];
    if ([_data valueForKey:@"subData"]) {
        detailLabel.frame = [[_data valueForKey:@"subData"][@"textRect"] CGRectValue];
        [detailLabel setText:[_data valueForKey:@"subData"][@"text"]];
        detailLabel.hidden = NO;
    }
}
```
这个方法自定了一个Label，在`setText`方法中同样是异步绘制内容，并且底层实现是使用`CoreText`性能方面也是比较优越

下边的代码我稍微做了精简
```Objective-C
- (void)setText:(NSString *)text{
    ....
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSString *temp = text;
        _text = text;
        CGSize size = self.frame.size;
        size.height += 10;
        UIGraphicsBeginImageContextWithOptions(size, ![self.backgroundColor isEqual:[UIColor clearColor]], 0);
        CGContextRef context = UIGraphicsGetCurrentContext();
        if (context==NULL) {
            return;
        }
        if (![self.backgroundColor isEqual:[UIColor clearColor]]) {
            [self.backgroundColor set];
            CGContextFillRect(context, CGRectMake(0, 0, size.width, size.height));
        }
        CGContextSetTextMatrix(context,CGAffineTransformIdentity);
        CGContextTranslateCTM(context,0,size.height);
        CGContextScaleCTM(context,1.0,-1.0);

        //Determine default text color
        UIColor* textColor = self.textColor;

        //Set line height, font, color and break mode
        CGFloat minimumLineHeight = self.font.pointSize,maximumLineHeight = minimumLineHeight, linespace = self.lineSpace;
        CTFontRef font = CTFontCreateWithName((__bridge CFStringRef)self.font.fontName, self.font.pointSize,NULL);
        CTLineBreakMode lineBreakMode = kCTLineBreakByWordWrapping;
        CTTextAlignment alignment = CTTextAlignmentFromUITextAlignment(self.textAlignment);
        //Apply paragraph settings
        CTParagraphStyleRef style = CTParagraphStyleCreate((CTParagraphStyleSetting[6]){
            {kCTParagraphStyleSpecifierAlignment, sizeof(alignment), &alignment},
            {kCTParagraphStyleSpecifierMinimumLineHeight,sizeof(minimumLineHeight),&minimumLineHeight},
            {kCTParagraphStyleSpecifierMaximumLineHeight,sizeof(maximumLineHeight),&maximumLineHeight},
            {kCTParagraphStyleSpecifierMaximumLineSpacing, sizeof(linespace), &linespace},
            {kCTParagraphStyleSpecifierMinimumLineSpacing, sizeof(linespace), &linespace},
            {kCTParagraphStyleSpecifierLineBreakMode,sizeof(CTLineBreakMode),&lineBreakMode}
        },6);

        NSDictionary* attributes = [NSDictionary dictionaryWithObjectsAndKeys:(__bridge id)font,(NSString*)kCTFontAttributeName,
                                    textColor.CGColor,kCTForegroundColorAttributeName,
                                    style,kCTParagraphStyleAttributeName,
                                    nil];

        //Create attributed string, with applied syntax highlighting
        NSMutableAttributedString *attributedStr = [[NSMutableAttributedString alloc] initWithString:text attributes:attributes];
        CFAttributedStringRef attributedString = (__bridge CFAttributedStringRef)[self highlightText:attributedStr];

        //Draw the frame
        CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((CFAttributedStringRef)attributedString);

        CGRect rect = CGRectMake(0, 5,(size.width),(size.height-5));

        if ([temp isEqualToString:text]) {
            [self drawFramesetter:framesetter attributedString:attributedStr textRange:CFRangeMake(0, text.length) inRect:rect context:context];
            CGContextSetTextMatrix(context,CGAffineTransformIdentity);
            CGContextTranslateCTM(context,0,size.height);
            CGContextScaleCTM(context,1.0,-1.0);
            UIImage *screenShotimage = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();
            dispatch_async(dispatch_get_main_queue(), ^{
                CFRelease(font);
                CFRelease(framesetter);
                CFRelease(style);
                [[attributedStr mutableString] setString:@""];

                if (drawFlag==flag) {
                    if (isHighlight) {
                      ....
                    } else {
                        if ([temp isEqualToString:text]) {
                            ...
                            highlightImageView.image = nil;
                            labelImageView.image = nil;
                            labelImageView.image = screenShotimage;
                        }
                    }
//                    [self debugDraw];//绘制可触摸区域
                }
            });
        }
    });
}
```
这个实现与`- (void)drawInContext:(CGContextRef)context withPosition:(CGPoint)p andFont:(UIFont *)font andTextColor:(UIColor *)color andHeight:(float)height`这个方法的实现很相似

接下来就是`[self loadThumb];`这个方法这个方法其实关键就是应用了`SDWebimage`加载图片，因为此框架在加载图片已经帮我优化了。（异步解压图片，并且将图片绘制到画布上，从而不会使解压数据在内存中驻留）
# 总结
作者通过以上三点确实做到了性能极大的提高。不过我在研究的过程中遇到了两个bug：
* 代码中有内存泄漏，主要是由于有一处`CTParagraphStyleRef style`这个对象没有释放
* 当我单手使劲向上滑动，界面停下的时候页面不会加载数据。其实就是由于监听了`- (void)scrollViewWillEndDragging:(UIScrollView *)scrollView withVelocity:(CGPoint)velocity targetContentOffset:(inout CGPoint *)targetContentOffset`在这个方法导致的

在以上三个优化点的基础上其实我们还可以做一个比较明显的优化就是将绘制结果用`NSCache`缓存起来，没必要每次都进行绘制。加上这个性能会有更大的提升

加上最后一点总共四点，除了第一点有的项目不会接受外，其他三点和`SDWebimage`的优化方式只要我们在项目适当的使用，基本上能够解决很大一部分的性能问题的
